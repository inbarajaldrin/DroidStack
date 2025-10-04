## **List All Prims in USD Scene**

```python
from pxr import Usd, UsdGeom
import omni.usd

def list_prims() -> str:
    """
    List all prim paths in the current USD scene.
    Useful for discovering available objects before moving, reading poses, or other operations.
    
    Example:
        list_prims()
    """
    try:
        # Get the current USD stage
        stage = omni.usd.get_context().get_stage()
        
        if not stage:
            return "Error: No stage is currently open"
        
        # Get all prims in the stage
        prims = []
        for prim in stage.Traverse():
            prims.append({
                'path': str(prim.GetPath()),
                'name': prim.GetName(),
                'type': prim.GetTypeName()
            })
        
        response = f"Prims in scene ({len(prims)}):\n\n"
        
        for prim in prims:
            path = prim['path']
            name = prim['name']
            prim_type = prim['type']
            
            # Format: path (type) or path [name] (type) if name differs
            if name and name != path.split('/')[-1]:
                response += f"{path} [{name}] ({prim_type})\n"
            else:
                response += f"{path} ({prim_type})\n"
        
        return response
            
    except Exception as e:
        print(f"Error listing prims: {str(e)}")
        return f"Error listing prims: {str(e)}"

# Call the function and print the result
print(list_prims())
```

## **List Prims with Properties (Position & Orientation)**

```python
from pxr import Usd, UsdGeom, Gf
import omni.usd
import math

def quaternion_to_euler_degrees(quat):
    """
    Convert quaternion (w, x, y, z) to Euler angles in degrees (roll, pitch, yaw)
    """
    w, x, y, z = quat
    
    # Roll (x-axis rotation)
    sinr_cosp = 2 * (w * x + y * z)
    cosr_cosp = 1 - 2 * (x * x + y * y)
    roll = math.atan2(sinr_cosp, cosr_cosp)
    
    # Pitch (y-axis rotation)
    sinp = 2 * (w * y - z * x)
    if abs(sinp) >= 1:
        pitch = math.copysign(math.pi / 2, sinp)
    else:
        pitch = math.asin(sinp)
    
    # Yaw (z-axis rotation)
    siny_cosp = 2 * (w * z + x * y)
    cosy_cosp = 1 - 2 * (y * y + z * z)
    yaw = math.atan2(siny_cosp, cosy_cosp)
    
    # Convert to degrees
    return [math.degrees(roll), math.degrees(pitch), math.degrees(yaw)]

def get_world_xforms_poses() -> str:
    """
    Get position and orientation of all Xform prims under /World (excluding system objects).
    Returns position and orientation in both quaternions and Euler angles (degrees).
    """
    try:
        # Get the current USD stage
        stage = omni.usd.get_context().get_stage()
        
        if not stage:
            return "Error: No stage is currently open"
        
        # Get the World prim
        world_prim = stage.GetPrimAtPath("/World")
        
        if not world_prim.IsValid():
            return "Error: /World prim not found"
        
        # Exclude system objects
        exclude_list = ["defaultGroundPlane", "Physics_Materials", "Environment"]
        
        response = "Poses of Xform prims under /World:\n\n"
        
        for child in world_prim.GetChildren():
            if child.GetTypeName() == "Xform" and child.GetName() not in exclude_list:
                path = str(child.GetPath())
                name = child.GetName()
                
                # Get position
                translate_attr = child.GetAttribute("xformOp:translate")
                position = [0.0, 0.0, 0.0]
                if translate_attr.IsValid():
                    translate_value = translate_attr.Get()
                    if translate_value:
                        position = [float(translate_value[0]), float(translate_value[1]), float(translate_value[2])]
                
                # Get orientation (quaternion)
                orient_attr = child.GetAttribute("xformOp:orient")
                quaternion = [1.0, 0.0, 0.0, 0.0]  # Default: w, x, y, z
                if orient_attr.IsValid():
                    orient_value = orient_attr.Get()
                    if orient_value:
                        quaternion = [
                            float(orient_value.GetReal()),
                            float(orient_value.GetImaginary()[0]),
                            float(orient_value.GetImaginary()[1]),
                            float(orient_value.GetImaginary()[2])
                        ]
                
                # Convert quaternion to Euler angles in degrees
                euler_degrees = quaternion_to_euler_degrees(quaternion)
                
                # Format output
                response += f"{name} ({path}):\n"
                response += f"  Position (x, y, z): [{position[0]:.4f}, {position[1]:.4f}, {position[2]:.4f}]\n"
                response += f"  Quaternion (w, x, y, z): [{quaternion[0]:.4f}, {quaternion[1]:.4f}, {quaternion[2]:.4f}, {quaternion[3]:.4f}]\n"
                response += f"  Euler (roll, pitch, yaw) in degrees: [{euler_degrees[0]:.2f}°, {euler_degrees[1]:.2f}°, {euler_degrees[2]:.2f}°]\n\n"
        
        return response
            
    except Exception as e:
        print(f"Error getting poses: {str(e)}")
        return f"Error getting poses: {str(e)}"

# Call the function and print the result
print(get_world_xforms_poses())
```
