import maya.cmds as cmds
import re

def get_lights_by_aov_group():
    lights = cmds.ls(type=['light', 'aiAreaLight', 'aiSkyDomeLight', 'aiPhotometricLight', 'aiMeshLight'])
    group_map = {}
    for light in lights:
        group = cmds.getAttr(f"{light}.aiAov") if cmds.attributeQuery("aiAov", node=light, exists=True) else ""
        if group:
            group_map.setdefault(group, []).append(light)
    return group_map

def get_or_create_group_cc(group_name, sample_light):
    cc_name = f"{group_name}_aiColorCorrect"
    existing_cc = cmds.listConnections(f"{sample_light}.color", source=True, destination=False, type='aiColorCorrect')
    if existing_cc:
        return existing_cc[0]
    if cmds.objExists(cc_name) and cmds.nodeType(cc_name) == 'aiColorCorrect':
        return cc_name
    cc = cmds.shadingNode('aiColorCorrect', asUtility=True, name=cc_name)
    orig_conn = cmds.listConnections(f"{sample_light}.color", plugs=True, source=True, destination=False) or []
    if orig_conn:
        cmds.disconnectAttr(orig_conn[0], f"{sample_light}.color")
        cmds.connectAttr(orig_conn[0], f"{cc}.input", force=True)
    cmds.connectAttr(f"{cc}.outColor", f"{sample_light}.color", force=True)
    return cc

def apply_color_grade_to_group(group_name, rgb):
    group_map = get_lights_by_aov_group()
    lights = group_map.get(group_name)
    if not lights:
        cmds.warning(f"No lights found in group: {group_name}")
        return
    any_mapped = any(cmds.listConnections(f"{light}.color", source=True) for light in lights)
    log_entries = []

    if not any_mapped:
        for light in lights:
            try:
                current = cmds.getAttr(f"{light}.color")[0]
                new_color = [current[i] * rgb[i] for i in range(3)]
                cmds.setAttr(f"{light}.color", *new_color, type='double3')
                log_entries.append(f"{light}: Existing Color: {current} | Applied: {rgb} | New Color: {new_color}")
            except Exception as e:
                cmds.warning(f"Failed to apply direct color to {light}: {e}")
    else:
        cc = get_or_create_group_cc(group_name, lights[0])
        for light in lights:
            upstream = cmds.listConnections(f"{light}.color", plugs=True, destination=False) or []
            if upstream:
                cmds.disconnectAttr(upstream[0], f"{light}.color")
            cmds.connectAttr(f"{cc}.outColor", f"{light}.color", force=True)
        current_mult = cmds.getAttr(f"{cc}.multiply")[0]
        new_mult = [current_mult[i] * rgb[i] for i in range(3)]
        cmds.setAttr(f"{cc}.multiply", *new_mult, type='double3')
        log_entries.append(f"{cc}: Existing RGB: {current_mult} | Applied: {rgb} | Output RGB: {new_mult}")

    log_view_ui.update_log(log_entries)
    cmds.inViewMessage(amg=f'<hl>Color Grade Applied</hl> to group <hl>{group_name}</hl>', pos='topCenter', fade=True)

def apply_exposure_to_group(group_name, exposure_value):
    group_map = get_lights_by_aov_group()
    if group_name not in group_map:
        cmds.warning(f"No lights found in group: {group_name}")
        return
    log_entries = []
    for light in group_map[group_name]:
        try:
            current_exposure = cmds.getAttr(f"{light}.aiExposure") if cmds.attributeQuery("aiExposure", node=light, exists=True) else 0.0
            new_exposure = current_exposure + exposure_value
            cmds.setAttr(f"{light}.aiExposure", new_exposure)
            log_entries.append(f"{light}: Existing Exposure: {current_exposure:.2f} | +{exposure_value:.2f} → {new_exposure:.2f}")
        except Exception as e:
            cmds.warning(f"Failed to process {light}: {e}")
    log_view_ui.update_log(log_entries)
    cmds.inViewMessage(amg=f'<hl>Exposure Applied</hl> to group <hl>{group_name}</hl>', pos='topCenter', fade=True)

def apply_saturation_to_group(group_name, saturation_value):
    group_map = get_lights_by_aov_group()
    lights = group_map.get(group_name)
    if not lights:
        cmds.warning(f"No lights found in group: {group_name}")
        return
    cc = get_or_create_group_cc(group_name, lights[0])
    try:
        cmds.setAttr(f"{cc}.saturation", saturation_value)
        log_view_ui.update_log([f"{cc}: Saturation set to {saturation_value:.3f}"])
        cmds.inViewMessage(amg=f'<hl>Saturation Applied</hl> to group <hl>{group_name}</hl>', pos='topCenter', fade=True)
    except Exception as e:
        cmds.warning(f"Failed to set saturation for {group_name}: {e}")

class LightGradeUI:
    def __init__(self):
        self.win = "arnoldLightRevertUI"
        self.aov_groups = []
        self.selected_group = ""
        self.rgb_input = [1.0, 1.0, 1.0]
        self.exposure_input = 0.0
        self.saturation_input = 1.0

    def show(self):
        if cmds.window(self.win, exists=True):
            cmds.deleteUI(self.win)
        self.aov_groups = list(get_lights_by_aov_group().keys())
        if not self.aov_groups:
            cmds.warning("No AOV groups found.")
            return
        self.selected_group = self.aov_groups[0]

        cmds.window(self.win, title="Arnold Light Revert", sizeable=True, widthHeight=(420, 650), backgroundColor=(0.18, 0.18, 0.18))
        cmds.columnLayout(adjustableColumn=True)
        cmds.separator(height=10)
        cmds.text(label="Created by Pramod G", align="center", font="smallPlainLabelFont")
        cmds.text(label="ArtStation: https://www.artstation.com/pramod_pro", align="center", hyperlink=True)
        cmds.text(label="LinkedIn: https://www.linkedin.com/in/pramod-g-38064a53/", align="center", hyperlink=True)
        cmds.separator(height=10)
        cmds.text(label="Arnold Light Revert v1.3", align="center", font="boldLabelFont", backgroundColor=(0.3, 0.3, 0.3))
        cmds.separator(height=10)

        cmds.optionMenu("aovGroupMenu", label="Select AOV Group", changeCommand=self.update_group)
        for grp in self.aov_groups:
            cmds.menuItem(label=grp)
        cmds.separator(height=10)

        cmds.frameLayout(label="Color Grading", collapsable=True, backgroundColor=(0.25, 0.25, 0.25))
        self.prev_rgb_display = cmds.text(label="Existing RGB: 1.000, 1.000, 1.000", backgroundColor=(0.15, 0.15, 0.15))
        cmds.rowLayout(numberOfColumns=2, adjustableColumn=1)
        self.input_field = cmds.colorSliderGrp(label="RGB Multiplier", rgb=(1, 1, 1), changeCommand=self.update_output)
        cmds.button(label="Paste RGB", width=80, command=self.paste_rgb_values, backgroundColor=(0.6, 0.6, 1.0))
        cmds.setParent('..')
        cmds.button(label="Apply Color Grade", command=self.apply_grade, backgroundColor=(0.5, 0.8, 0.5))
        cmds.setParent('..')

        cmds.frameLayout(label="Exposure Control", collapsable=True, backgroundColor=(0.25, 0.25, 0.25))
        self.prev_exposure_display = cmds.text(label="Existing Exposure: 0.000", backgroundColor=(0.15, 0.15, 0.15))
        self.exposure_input_field = cmds.textFieldGrp(label="Exposure Change", text="0.0", changeCommand=self.update_output)
        self.output_exposure_display = cmds.text(label="New Exposure: 0.00", backgroundColor=(0.15, 0.15, 0.15))
        cmds.button(label="Apply Exposure", command=self.apply_exposure, backgroundColor=(0.5, 0.8, 0.5))
        cmds.setParent('..')

        cmds.frameLayout(label="Saturation Control", collapsable=True, backgroundColor=(0.25, 0.25, 0.25))
        self.saturation_slider = cmds.floatSliderGrp(label="Saturation", field=True, minValue=0.0, maxValue=2.0, value=1.0, step=0.01)
        cmds.button(label="Apply Saturation", command=self.apply_saturation, backgroundColor=(0.8, 0.7, 0.4))
        cmds.setParent('..')

        cmds.frameLayout(label="Action Log", collapsable=True, backgroundColor=(0.25, 0.25, 0.25))
        self.log_display = cmds.scrollField(editable=False, wordWrap=True, height=120, backgroundColor=(0.1, 0.1, 0.1))
        cmds.setParent('..')

        cmds.button(label="Refresh", command=self.refresh_ui, backgroundColor=(0.3, 0.6, 1.0))
        cmds.separator(height=5)
        cmds.button(label="Clear Log", command=self.clear_log, backgroundColor=(0.7, 0.3, 0.3))
        cmds.showWindow(self.win)
        self.update_display()

    def update_group(self, group):
        self.selected_group = group
        self.update_display()

    def update_display(self):
        lights = get_lights_by_aov_group().get(self.selected_group, [])
        if not lights:
            return
        sample = lights[0]
        upstream_cc = cmds.listConnections(f"{sample}.color", source=True, destination=False, type='aiColorCorrect')
        if upstream_cc:
            prev_rgb = cmds.getAttr(f"{upstream_cc[0]}.multiply")[0]
        else:
            named_cc = f"{self.selected_group}_aiColorCorrect"
            if cmds.objExists(named_cc) and cmds.nodeType(named_cc) == 'aiColorCorrect':
                prev_rgb = cmds.getAttr(f"{named_cc}.multiply")[0]
            else:
                prev_rgb = cmds.getAttr(f"{sample}.color")[0]
        cmds.text(self.prev_rgb_display, edit=True, label=f"Existing RGB: {prev_rgb[0]:.3f}, {prev_rgb[1]:.3f}, {prev_rgb[2]:.3f}")
        try:
            prev_exp = cmds.getAttr(f"{sample}.aiExposure") if cmds.attributeQuery("aiExposure", node=sample, exists=True) else 0.0
        except:
            prev_exp = 0.0
        cmds.text(self.prev_exposure_display, edit=True, label=f"Existing Exposure: {prev_exp:.3f}")
        cmds.textFieldGrp(self.exposure_input_field, edit=True, text=str(prev_exp))
        self.update_output()

    def update_output(self, *_):
        rgb = cmds.colorSliderGrp(self.input_field, query=True, rgb=True)
        exp_text = cmds.textFieldGrp(self.exposure_input_field, query=True, text=True)
        try:
            exp = float(exp_text)
        except:
            exp = 0.0
        cmds.text(self.output_exposure_display, edit=True, label=f"New Exposure: {exp:.2f}")

    def apply_grade(self, *_):
        rgb = cmds.colorSliderGrp(self.input_field, query=True, rgb=True)
        apply_color_grade_to_group(self.selected_group, rgb)

    def apply_exposure(self, *_):
        try:
            exp = float(cmds.textFieldGrp(self.exposure_input_field, query=True, text=True))
        except:
            exp = 0.0
        apply_exposure_to_group(self.selected_group, exp)

    def apply_saturation(self, *_):
        value = cmds.floatSliderGrp(self.saturation_slider, query=True, value=True)
        apply_saturation_to_group(self.selected_group, value)

    def paste_rgb_values(self, *_):
        if cmds.promptDialog(
            title='Paste RGB',
            message='Enter RGB (e.g. 1.0,0.8,0.6):',
            button=['OK', 'Cancel'], defaultButton='OK',
            cancelButton='Cancel', dismissString='Cancel'
        ) == 'OK':
            txt = cmds.promptDialog(query=True, text=True)
            vals = re.findall(r"[\d.]+", txt)
            if len(vals) >= 3:
                rgb = [min(max(float(vals[i]), 0.0), 100.0) for i in range(3)]
                cmds.colorSliderGrp(self.input_field, edit=True, rgb=rgb)
                self.update_output()
            else:
                cmds.warning("Could not parse 3 RGB values.")

    def update_log(self, entries):
        current = cmds.scrollField(self.log_display, query=True, text=True)
        cmds.scrollField(self.log_display, edit=True, text=current + "\n" + "\n".join(entries))

    def clear_log(self, *_):
        cmds.scrollField(self.log_display, edit=True, text="")

    def refresh_ui(self, *_):
        cmds.colorSliderGrp(self.input_field, edit=True, rgb=(1, 1, 1))
        cmds.textFieldGrp(self.exposure_input_field, edit=True, text="0.0")
        cmds.floatSliderGrp(self.saturation_slider, edit=True, value=1.0)
        self.update_display()

# Launch the UI
log_view_ui = LightGradeUI()
log_view_ui.show()
