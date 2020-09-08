# Overview

In USD it's possible to define arbitrary data for a primitive that can be referenced in material graphs for any sort of calculations.
This data can vary across the primitive in different ways (i.e. data interpolation):

* per vertex - values defined for each vertex (position) - `vertex`
* per face vertex - values defined for each vertex of each face - `faceVarying`
* per face - one value for each face - `uniform`
* per primitive - one value for primitive - `constant`


This data is typically identified by name and called primvar.
Primvar can be seen as geometry's data export socket that can be connected to the material. 

## High-Level Example

Trivial combination of geometry primitive and material primitive.

### Material 

Let's say our material has `color` property.
Instead of explicitly setting `color` value (to vec4/to texture/etc) we specify that `color` value is imported from the geometry that uses this material. For example, the name of the primvar used here is "rgbnoise".
The material does not care how exactly a particular geometry calculates (exports) this value for each ray hit point.
The material only declares that some primvar should be used here or there.
If a particular geometry does not have a required primvar, we use some fallback value.

### Geometry

Let's say our geometry primitive is a triangle mesh.
This mesh has points, indices, and primvar of vec3 array type named "rgbnoise" with vertex interpolation (one vec3 value for each point).
So this geometry should define an export socket with "rgbnoise" name.
In this particular case (triangle mesh, vertex interpolation), the exported value can be calculated via barycentric interpolation of three values that are taken from the points of the current triangle.

# API proposal

* New rpr_material_node_type: RPR_MATERIAL_NODE_PRIMVAR
* New rpr_material_node_input: RPR_MATERIAL_INPUT_ID, RPR_MATERIAL_INPUT_FALLBACK
* New function to set primvar id: rprMaterialNodeSetInputStringByKey(rpr_material_node, rpr_material_node_input, const char*)
* New enum for primvar interpolation:
    enum rpr_primvar_interpolation_type {
        RPR_PRIMVAR_INTERPOLATION_CONSTANT,
        RPR_PRIMVAR_INTERPOLATION_UNIFORM,
        RPR_PRIMVAR_INTERPOLATION_VERTEX,
        RPR_PRIMVAR_INTERPOLATION_FACE_VARYING,
    }
* New function to set shape primvar: rprShapeSetPrimvar(rpr_shape, const char* id, rpr_primvar_interpolation_type, rpr_buffer data)

## Example

### Material

```
const char* primvarId = "rgbnoise";

rpr_material_node diffuseNode;
/* Create diffuse node */

// Create primvar node
rpr_material_node primvarNode;
rprMaterialSystemreateNode(matSys, RPR_MATERIAL_NODE_PRIMVAR, &primvarNode);

// Set primvar node inputs
rprMaterialNodeSetInputStringByKey(primvarNode, RPR_MATERIAL_INPUT_ID, primvarId);
rprMaterialNodeSetInputFByKey(primvarNode, RPR_MATERIAL_INPUT_FALLBACK, 1.0f, 0.0f, 1.0f, 1.0f); // fallback to pink

// Use primvar node as color property
rprMaterialNodeSetInputNByKey(diffuseNode, RPR_MATERIAL_INPUT_COLOR, primvarNode);

```

### Geometry

```
rpr_shape mesh;
/* Create mesh */

const int numPoints;
std::vector<float> rgbNoise;
for (int i = 0; i < numPoints; ++i) {
	for (int j = 0; j < 3; ++j) {
		rgbNoise.push_back(getNoise(i * 3 + j));
	}
}

// Create primvar rpr_buffer
rpr_buffer_desc bufferDesc = {};
bufferDesc.nb_element = numPoints;
bufferDesc.element_type = RPR_BUFFER_ELEMENT_TYPE_FLOAT32;
bufferDesc.element_channel_size = 3;
rpr_buffer buffer;
rprContextCreateBuffer(rprContext, &bufferDesc, rgbNoise.data(), &buffer);

rprShapeSetPrimvar(mesh, primvarId, RPR_PRIMVAR_INTERPOLATION_VERTEX, buffer);
```
