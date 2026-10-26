# Hand URDF models (required for FK / RViz)

Place the official LinkerHand URDF (with meshes) here:

```
data/urdf/{model}/{side}/linkerhand_{model}_{side}.urdf
```

Versioned filenames are also accepted and resolved to the stable alias,
e.g. `linkerhand_l6v3.1_left.urdf` satisfies a request for `l6` / `left`.

Override root: `export LINKERHAND_URDF_DIR=/path/to/linkerhand-urdf`

Search order: `data/urdf` → `LINKERHAND_URDF_DIR` → installed package share → `~/{gcl_ws,gitea_ws,ros_ws,cad_ws}/.../linkerhand-urdf` → `/opt/linkerhand/urdf`

Nodes may also take an explicit `urdf_path` parameter, which bypasses the search.

## Bundled models in installed deployments

The repository includes the O6 **right** URDF and its meshes. Wheels and colcon
installations preserve this tree under `share/hand_gesture_primitives/urdf`;
source distributions carry the same files under their project-local `data/urdf`.
There is no second resource tree in the source checkout. The isolated Jazzy
installer copies and hashes these assets along with the Python and message packages.

The resolver checks the repository source tree, `LINKERHAND_URDF_DIR`, the ament
package share directory, and `sys.prefix/share/hand_gesture_primitives/urdf`, then
the existing external workspace paths. An explicit node `urdf_path` takes priority.

Left O6 and the other models must be supplied separately; the right model is not
mirrored or substituted for another hand. Configure external models with:

```bash
export LINKERHAND_URDF_DIR=/path/to/linkerhand-urdf
```

Pinocchio remains an optional FK dependency. In the isolated Jazzy environment,
use the ROS/system package rather than replacing apt NumPy/SciPy with pip builds.

## Without a URDF

FK degrades rather than breaking gesture control:

- `gesture_node` — all gesture primitives still run; fingertip topic/markers are not published
- `joint_state_bridge` — exits with an error (its only job is driving the mesh model via FK)
