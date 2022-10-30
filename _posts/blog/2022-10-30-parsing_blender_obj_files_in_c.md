---

layout: post
title: Parsing Blender Obj Files with C
author: Kyle Cooper
category: blog 
excerpt_separator: <!--more-->

---

The most basic method of parsing a obj file exported from blender. This guide uses C/C++ without any additional libraries. We try to stick with raw types. For example most guides will have you put the vertices in a a Vector3\<Float\>. In this guide we are doing both. We are parsing into a custom vf3 and then flattening them into an array and use a stride of 3. 
<!--more-->


For this parser, we only care about the basic structure, normals and texture coordinates of the object. 

Our vf3 (or vector of 3 floats looks like the following).
```c++
typedef struct vf3
{
    union
    {
        struct
        {
            f32 x;
            f32 y; 
            f32 z; 
        };
        f32 e[3];
    };
} vf3;
```

The goal is to render this in our 'renderer' using OpenGL. We skip over the OpenGL instructions in this doc. Essentially you need an index buffer and attribArray.

## Exporting The Blender Obj File
First, open Blender. By default it should just render a cube: 
![blender default cube](https://firebasestorage.googleapis.com/v0/b/blog-3ac4f.appspot.com/o/blender_opengl_post%2Fblender_cube.png?alt=media&token=7305b763-dd28-461b-8e62-9bbe9f73d743)
Then go to File > Export. Select WaveFront(.obj). A settings menu will appear.
Expand Geometry and select 'Triangulate Faces', this is important as our cube will not render correctly. Blender assumes we render Quads when exporting but we want to render with Triangles. 

You will get something like the following. **This is not the actual file, I have truncated most of the data**
```obj
# Blender v2.93.0 OBJ File: ''
# www.blender.org
mtllib triangulate-faces.mtl
o Cube
v 1.000000 1.000000 -1.000000
...
vt 0.875000 0.500000
...
vn 0.0000 1.0000 0.0000
...
usemtl Material
s off
f 5/1/1 3/2/1 1/3/1 
...
```
The .obj file describes the structure of the exported object, another file with a `.mtl` extension describes the materials, we can ignore this for now. 

The following is a cheat sheet of what the above means; 
- `o`  describes the object name we want to create
- `v`  the vertices of the object. Notices how there are fewer than the total vertices because we use index buffering as most of the vertices are repeated
- `vt`  these are our `uv` s or texture coordinates for the object
- `s`  can be ignored, has to do with shading
- `usemtl` can be ignored
- `f`  these are our indices, if you did not export from blender with 'Triangulate Faces' you will have a shorter list here, thus causing OpenGL to not render correctly. The layout is as follows: 
	- The index pattern is ```f v/vt/vn``` where v, vt and vn are the index to render for the specific type. So for V we render vertices in that order. vt is our texture coordinates and vn is our normal  Ideally we want to parse out these into an array of indices so `[0,1,3,5,6]` 
	- The f stands for 'Face' or a triangle face this is essentially a list of how the object goes together. We use this kind of list, rather than all the vertices because we generally have lots of repetition in our vertices so this helps save space. Its easy to see when you look at how many vertices you need to specify if you 'hand write' out all the vertices for a cube.

## The Crude Parser

1. We need to load the file. In my case I have a custom built file reader, but really it uses the OSes [ReadFile](https://learn.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-readfile) in the fileapi.h file
2. Once we have the raw contents You can start parsing. 

**You may need to make this 'guide' fit to your use case**

**Read the contents of the file into a struct called LoadObj**. You can change this to suite your needs because we are really just checking to make sure we read the file.
```c++
typedef struct LoadedObj
{
    char *fileName;
    char *contents;
    ui32 size;
} LoadedObj;

static LoadedObj 
LoadObjFile( DebugReadEntireFileType *ReadFileContents,  
        char *fileName)
{
    LoadedObj result = {};
	// This is where you want to read the file with w.e. function you have
    debug_file_data_from_read readContents = 
        ReadFileContents(threadContext, fileName); 

    Assert(readContents.size > 0); //We could not read the file, you can fail more gracefully
    if (readContents.size > 0) 
    {
        result.contents = (char *)readContents.contents;
        result.size = readContents.size;
    }
    return result;
}
```

**Start parsing**
Here is the mesh structure we use to store our mesh
``` C++
typedef struct MeshStructureAttributes
{
    LoadedObj *asset; // reference to the obj, if ytou need
    
    ui32* indices; //our vertex indicies
    f32* verticies; // the vertexes in an array 
    vf3* textures;
    vf3* norms;
    ui32* faces;

	// total counts for each of the above
    ui32 vertCount;
    ui32 textureCount;
    ui32 normalCount;
    ui32 faceCount;

	//keep stride info if you need, really this is usually 3 anyway 
    ui32 faceStride = 3;
    // The # of indicies in the faces, that represent a vert
    ui32 indiciesCountForVerts;
} MeshStructureAttributes;
```

Below is my 'controller' function for parsing out the file. It calls out to other custom functions. 
I am not using any libraries here.
```C++
static staticMeshStructureAttributes*
ParseObjFileContents(memory_arena *memArena, LoadedObj meshObjFile)
{
    MeshStructureAttributes *result = PushStruct(memArena, MeshStructureAttributes);
    char *contents = meshObjFile.contents;
    Assert(contents > 0); // TODO we will want to fail more gracefully 
    if (contents != 0) 
    {
        char v = 'v';
        char space =  ' ';
        char t =  't';
        char n =  'n';
        char f =  'f';

        ui32 firstVertIndex = 0;
        ui32 firstTextureIndex = 0;
        ui32 firstNormalIndex = 0; 
        ui32 firstFaceIndex = 0;

        ui32 faceRows = 0;

        ui32 position = 0;
		// Within this while loop our goal is to find the lengths of each type. As 
		// well as the first index of where the types reside 
        while (*contents++)
        {
			// You may want to build in some protection in case you are worried 
			// about out of bounds issues
            char charAtCurrent = *contents;
			// we sometimes want to look at the next char, for instance v and space to 
			// indicate a vertex
            char charAtNext = *(contents + 1);
            if (charAtNext == '\0')
            {
			// Ideally we should never get here otherwise we need better checking
                Assert(
                        charAtCurrent > 0 && 
                        charAtCurrent < 127  && 
                        charAtCurrent == '\n'
                        );
                break;
            }
            // NOTE the position + 3 is determined by the spaces and 
            // chars in the obj file + 1
            if (charAtCurrent == v && charAtNext == space)
            {
                if (firstVertIndex == 0) 
                { 
                    firstVertIndex = position + 3;
                }
                result->vertCount++;
            }
            if (charAtCurrent == v && charAtNext == t)
            {
                if (firstTextureIndex == 0) 
                { 
                    firstTextureIndex = position + 4;
                }
                result->textureCount++;
            }
            if (charAtCurrent == v && charAtNext == n)
            {
                if (firstNormalIndex ==  0) 
                { 
                    firstNormalIndex = position + 4;
                }
                result->normalCount++;
            }
            if (charAtCurrent == f && charAtNext == space)
            {
                if (firstFaceIndex == 0) 
                { 
                    firstFaceIndex = position + 3;
                }
                faceRows++;
            }
            position++;
        }

        // face count is rows * columns where columns is 3 * 4 because 
        // we group our textures in 3s with 4 in each row. Essentially you 
        // look at the rows of f 1/2/3
		// becarful here though because this may not be true for all obj files, 
		// change as you need!
        result->faceCount = faceRows*(3*4);

		// Here is where you allocate (malloc if you need). See my notes on memory 
		// below
        result->verts = PushVerts(memArena, result->vertCount, vf3);
        result->norms = PushNormals(memArena, result->normalCount, vf3);
        result->textures = PushTextures(memArena, result->textureCount, vf3);
        result->faces = PushFaces(memArena, result->faceCount, ui32);

		// Now we parse our verts as need 
        {
	        // Here is where we create out vertex for a vf3* of the vertices
	        // we could go directly to a list right here if you need
            contents = meshObjFile.contents;
            contents += firstVertIndex; 
            char* verts = contents; 
            ParseVertsFromLoadedObjToVF3AndAppend(verts, result->verts, result->vertCount, 3);
        }
        {
            contents = meshObjFile.contents;
            contents += firstTextureIndex; 
            char* texture = contents; 
            ParseVertsFromLoadedObjToVF3AndAppend(texture, result->textures, result->textureCount, 2);
        }
        {
            contents = meshObjFile.contents;
            contents += firstNormalIndex; 
            char* norms = contents; 
            ParseVertsFromLoadedObjToVF3AndAppend(norms, result->norms, result->normalCount, 3);
        }
        {
			// Here is where we go through our faces or indicies
            contents = meshObjFile.contents;
            contents += firstFaceIndex; 
            char* faces = contents; 

            ui32 facePosition = 0;
            // Count should be row * columns of faces within 
            // the groupings of 3 ex. f 3/4/5
            for (ui32 current = 0; 
                    current < result->faceCount;
                    ++current)
            { 
                i32 num = 0;
                b32 updated = false;
				// since we want to only get the uints from the face indicies (not f 
				// or ' ' or / etc) we need to check if the number is a num
                while (IsNum(*faces))
                { 
					// We can convert the num we found to a number 
                    num = num * 10 + (i32)*faces - '0';
                    faces++;
                    updated = true; 
                } 
                if (updated == true) 
                {
                    result->faces[facePosition++] = num;
                }
                while (*faces && !IsNum(*faces)) 
                {
                    faces++;
                }
            }
        }
    }

	// The next function will flatten our indicies and verts as we need
    result->indices = FlattenIndiciesFromObjMesh(memArena, result);
	// With this function we flatten the vf3 array we created earlier. This might 
	// not be the most efficent way of creating this array if you ONLY want this 
	// type of array
    result->verticies = FlattenVerts(memArena, result);
    return result;
}
```

**Our ParseVertsFromLoadedObjToVF3 looks like the following:** This will convert the given array into an array of vf3s. If you want the flatten array and not an array of vf3s you can skip this step and instead just create the flat array.
```C++
inline void 
ParseVertsFromLoadedObjToVF3AndAppend(
        char* contentAtStartOfIndex,
        vf3 *arrayOfVertsToAppendTo,
        ui32 totalVertCount,
        ui32 floatsPerVert
        )
{
    Assert(floatsPerVert <= 3);
    char* verts = contentAtStartOfIndex; 
    for (ui32 currentVert = 0; 
            currentVert < totalVertCount;
            ++currentVert)
    { 
        f32 floatsPer[3] = {};
        for (ui32 currentVf3 = 0; 
                currentVf3 < floatsPerVert; 
                ++currentVf3)
        {
			// StringToFloat is a similar function to atof
            f32 val = StringToFloat(verts);
            floatsPer[currentVf3] = val;
            while (*verts != ' ')  
            {
                verts++;
            }
            verts++;
        }
        vf3 completeVert = {floatsPer[0], floatsPer[1], floatsPer[2]};
        arrayOfVertsToAppendTo[currentVert] = completeVert;
    }
}
```

**In this approach I flatten the vf3 array** back into a flat array. You could probably skip this step  OR head head directly to this step if you just want the flatten array
```c++
internal f32* 
FlattenVerts(memory_arena *memArena, MeshStructureAttributes *meshData) 
{
    vf3* verts = meshData->verts;
    ui32 vertCount = meshData->vertCount * 3;
    f32* result = PushArray(memArena, vertCount, f32);
    ui32 currentVert = 0;
    ui32 v = 0;
    while (currentVert < vertCount) 
    {
        vf3 *vert = verts + currentVert;
        result[v] = vert->x;
        result[++v] = vert->y;
        result[++v] = vert->z;
        ++v;
        ++currentVert;
    }
    return result;
}
```

You should not have an various arrays of data to use as you need


### Memory Allocation 
You can use whatever you fell comfortable with. I use a 'memory arena' to push onto my pre-allocated chunk of memory. Where I have `Push[XXX]` you could replace with something like malloc, and manage the memory how you please. 


## Areas for Improvement and Future Goals
1. Ideally, we would just forgo any method of putting this into an array of vf3 and just use a array of floats for the vertex
	1. With this we would also pack the vertex array, normals and texture cords into a single array. Then just use AttribArray in Opengl and give an offset for each vertex
1. Not all blender obj files are the same and we may want to account for this, but we are not looking for this kind of solution here
2. There are potentially some performance improvements that we could do. However, the best would be to binarize the data and create our on assets pack. After all, these assets while be a 'static' part of our game

## References
- [Wikipedia](https://en.wikipedia.org/wiki/Wavefront_.obj_file)
- [Blender OBJ docs](https://docs.blender.org/manual/en/2.79/addons/io_obj.html)
