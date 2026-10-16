# Qudi Migration to PySide6 Environment

Oct 16, 2026 · @Sam

## Overview

Qudi now runs from `C:\Users\SPUD1\Documents\workspace_sam\.venv` with the config `qudi-configs\SPUD202609.cfg`. The old setup in `experiment_workspace\experiment-main` used PySide2. The new one uses PySide6, so old custom modules must be ported before they load.

| Component | Old (experiment-main) | New (workspace\_sam) |
| --- | --- | --- |
| Python | 3.10.11 | 3.11.9 |
| Qt bindings | PySide2 5.15.2.1 | PySide6 6.11.0 (pinned) |
| qudi-core | 1.6.0 | 1.8.0 |
| qudi-iqo-modules | 0.6.0.dev0 | 1.0.0.dev0 (editable, `workspace_sam\qudi-iqo-modules`) |
| pyserial | 3.5 | 3.5 |
| Jupyter kernel | `qudi` | `qudi` (re-registered to the new venv) |

## Environment fixes

The venv needs two package changes: PySide6 held at 6.11.0, and `pyserial` in place of the wrong `serial` package. Run all commands from `C:\Users\SPUD1\Documents\workspace_sam` with qudi closed.

### Pin PySide6 to 6.11.0

PySide6 6.12.0 has a bug that blanks the scanner GUI's range, target and resolution boxes. It hits spinboxes that hide their arrow buttons when a stylesheet such as `qdark.qss` styles them; the text field shrinks to 1 px. Versions 6.10.1 and 6.11.0 render correctly.

PySide6-Pdf and PySide6-WebEngine exist only from 6.12 on (in 6.11 they ship inside Addons), so remove them first:

```
.venv\Scripts\python.exe -m pip uninstall -y PySide6-Pdf PySide6-WebEngine
.venv\Scripts\python.exe -m pip install PySide6==6.11.0 PySide6-Essentials==6.11.0 PySide6-Addons==6.11.0
```

qudi-core 1.8.0 only requires `PySide6>=6.10.1`, so a later `pip install -U` can bring 6.12 back. Add `PySide6<6.12` to any install notes or requirements file.

### Replace `serial` with `pyserial`

The PyPI package `serial` is a JSON/YAML serialization library, not the serial-port library. Both import as `serial`, so the laser failed with `AttributeError: module 'serial' has no attribute 'Serial'`.

```
.venv\Scripts\python.exe -m pip uninstall -y serial
.venv\Scripts\python.exe -m pip install pyserial==3.5
```

### Check the result

```
.venv\Scripts\python.exe -m pip list | findstr -i "pyside shiboken serial"
```

Expected: PySide6, PySide6\_Addons, PySide6\_Essentials and shiboken6 at 6.11.0, and pyserial 3.5.

## Porting old qudi code to PySide6

An old module needs more than an import change. Some Qt5 names were moved or removed in Qt6, and several of those errors appear only when a window opens.

| Qt5 / PySide2 code | PySide6 replacement | When it fails |
| --- | --- | --- |
| `from PySide2 import QtCore, ...` | `from PySide6 import QtCore, ...` | On import |
| `QtWidgets.QAction(...)` | `QtGui.QAction(...)` | When the GUI is built |
| `layout.SetFixedSize` (enum read through an object) | `QtWidgets.QLayout.SizeConstraint.SetFixedSize` | When the window is built |
| `QtWidgets.QDockWidget.AllDockWidgetFeatures` | `DockWidgetClosable \| DockWidgetMovable \| DockWidgetFloatable` (each under `QtWidgets.QDockWidget.DockWidgetFeature`) | On module activation |

Short enum names reached through the class, such as `QtCore.Qt.QueuedConnection` or `QtCore.Qt.AlignTop`, still work in PySide6 6.11.

### DLnsec laser files

The DLnsec laser modules are lab additions, not part of upstream qudi-iqo-modules (git shows them as untracked). Three were changed:

| File | Changes |
| --- | --- |
| `src/qudi/logic/laser_logic_dlnsec.py` | Import → PySide6 |
| `src/qudi/gui/laser/laser_control_dockwidget_dlnsec.py` | Import → PySide6 |
| `src/qudi/gui/laser/laser_gui_dlnsec.py` | Import → PySide6; 5 × `QAction` → `QtGui`; `SetFixedSize` enum; `AllDockWidgetFeatures` |

The hardware files `dlnsec_laser.py` and `dlnsec_laser_qudi.py` and the interface `simple_laser_interface_dlnsec.py` needed no Qt changes. The copies in `experiment_workspace` are the old PySide2 versions; do not copy them back.

### Checking another old module

1. Search the module for `PySide2` and change the import.
2. Search for `QtWidgets.QAction`, `exec_`, `QRegExp` and `QDesktopWidget`, and for enum values read through an object.
3. Check that every `QtCore.` / `QtWidgets.` / `QtGui.` name exists in PySide6, for example `hasattr(QtWidgets.QDockWidget, 'AllDockWidgetFeatures')` in the venv's Python.
4. Activate the module and open its GUI. Some errors appear only at that point.

- [ ] Port the magnet modules, which still import PySide2: `logic/magnet_logic.py`, `logic/magnet_data_logic.py`, `gui/magnet/magnetgui.py`, `gui/magnet/axes_control_dockwidget.py`.

## Scanning probe logic

`src/qudi/logic/scanning_probe_logic.py` is back to the committed upstream version; `git status` in `qudi-iqo-modules` is clean for it. An old PySide2 copy had been swapped in to debug the blank scanner GUI. It did not help, because the cause was the PySide6 6.12 bug above, not the logic.

To debug a module, keep the committed version and compare against the old one with `git diff` instead of swapping files. To restore a stock module: `git checkout -- <path>` in `qudi-iqo-modules`.

## Jupyter qudi kernel

The `qudi` notebook kernel does not start qudi. It connects to a running qudi on `localhost:18861` (`namespace_server_port` in `SPUD202609.cfg`) and exposes qudi's active modules by name, e.g. `dlnsec_laser`.

The kernel is registered at `C:\Users\SPUD1\AppData\Roaming\jupyter\kernels\qudi\kernel.json` and points to `workspace_sam\.venv\Scripts\python.exe`. To re-register it after rebuilding the venv, run `.venv\Scripts\qudi-install-kernel.exe`.

### Running a notebook

1. Start qudi from the new venv:

   ```
   C:\Users\SPUD1\Documents\workspace_sam\.venv\Scripts\qudi.exe -c C:\Users\SPUD1\Documents\workspace_sam\qudi-configs\SPUD202609.cfg
   ```
2. In the qudi manager, activate the modules the notebook uses (e.g. `dlnsec_laser`).
3. In VS Code, open the notebook: Command Palette → **Notebook: Select Notebook Kernel** → **Select Another Kernel** → **Jupyter Kernel** → **qudi**.
4. Confirm the environment in a cell:

   ```python
   import sys; sys.executable
   ```

   Expected: `'C:\\Users\\SPUD1\\Documents\\workspace_sam\\.venv\\Scripts\\python.exe'`. The kernel picker shows this as `qudi (Python 3.11.9) .venv\Scripts\python.exe`.

### Stale kernel path in VS Code

VS Code's Jupyter extension caches kernels per workspace. After the move it kept showing the old `experiment-main\.venv` path even though `kernel.json` was correct. **Developer: Reload Window** in the Command Palette, then reselecting the kernel, cleared it. Python 3.10.11 in the picker means the old environment; 3.11.9 means the new one.

## Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| `ModuleNotFoundError: No module named 'PySide2'` on activation | Module written for the old PySide2 setup | Change import to PySide6, then check the porting table |
| Scanner GUI range, target and resolution boxes are blank | PySide6 6.12.0 spinbox bug with `qdark.qss` | Pin PySide6 to 6.11.0 |
| `module 'serial' has no attribute 'Serial'` | Wrong PyPI package `serial` installed | Uninstall `serial`, install `pyserial==3.5`, restart qudi |
| `'QVBoxLayout' object has no attribute 'SetFixedSize'` | Qt6 enum read through an object | `QtWidgets.QLayout.SizeConstraint.SetFixedSize` |
| `QDockWidget has no attribute 'AllDockWidgetFeatures'` | Removed in Qt6 | OR the three `DockWidgetFeature` flags |
| VS Code kernel shows `experiment-main\.venv` path | Stale Jupyter extension cache | Developer: Reload Window, reselect `qudi` |
| `ConnectionRefusedError: [WinError 10061]` starting the kernel | Qudi not running, nothing on port 18861 | Start qudi with `SPUD202609.cfg`, then restart the kernel |
| Notebook name such as `dlnsec_laser` is undefined | Module not active in qudi | Activate it in the qudi manager |

After any `pip` change, restart qudi: a running qudi keeps the old packages loaded.
