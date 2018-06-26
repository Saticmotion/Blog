---
layout: default
title: Generating UI meshes in Unity
---

Some days ago I started working on a graphing library for Unity3D. For the rendering side of things I opted to generate a mesh for Unity's UI system to draw. Generating UI meshes happens in the `OnPopulateMesh` callback. As a parameter, you receive a `VertexHelper` which routes the mesh data to the correct underlying native code. `VertexHelper` is unfortunately quite poorly documented. With this blog post I hope to shed some light on how it works.

First of all, the Unity UI system is open source and `VertexHelper` can be found in the [Bitbucket repository](https://bitbucket.org/Unity-Technologies/ui/src/a3f89d5f7d145e4b6fa11cf9f2de768fea2c500f/UnityEngine.UI/UI/Core/Utility/VertexHelper.cs).

The public API of VertexHelper looks like this:
```
public void Clear()
public int currentVertCount {get;}
public int currentIndexCount {get;}
public void PopulateUIVertex(ref UIVertex vertex, int i)
public void SetUIVertex(UIVertex vertex, int i)
public void FillMesh(Mesh mesh)
public void Dispose()
public void AddVert(Vector3 position, Color32 color, Vector2 uv0, Vector2 uv1, Vector3 normal, Vector4 tangent)
public void AddVert(Vector3 position, Color32 color, Vector2 uv0)
public void AddVert(UIVertex v)
public void AddTriangle(int idx0, int idx1, int idx2)
public void AddUIVertexQuad(UIVertex[] verts)
public void AddUIVertexStream(List<UIVertex> verts, List<int> indices)
public void AddUIVertexTriangleStream(List<UIVertex> verts)
public void GetUIVertexStream(List<UIVertex> stream)
```

Simple Geometry
===

Most of the functions in the API are pretty simple, and do exactly what they say. If you only want to render a couple of quads, or some simple geometry, it will suffice to use the following set of functions:

```
public void Clear()
public void PopulateUIVertex(ref UIVertex vertex, int i)
public void SetUIVertex(UIVertex vertex, int i)
public void Dispose()
public void AddVert(Vector3 position, Color32 color, Vector2 uv0, Vector2 uv1, Vector3 normal, Vector4 tangent)
public void AddVert(Vector3 position, Color32 color, Vector2 uv0)
public void AddVert(UIVertex v)
public void AddTriangle(int idx0, int idx1, int idx2)
public void AddUIVertexQuad(UIVertex[] verts)
```

Some notes here:
- Any time a `UIVertex` is needed, it's enough to set a position and a color. All other data has reasonable defaults.
- `Clear` clears out all internal buffers left over from a previous frame. You will probably want to call this at the start of each `OnPopulateMesh`.
- `Dispose` cleans up all private resources. It's not needed to call this inside of `OnPopulateMesh`.
- `PopulateUIVertex` and `SetUIVertex` respectively get and set an existing vertex. Useful for doing simple displacement effects on simple geometry.
- `AddVert`, `AddUIVertexQuad` are useful for generating simple geometry. `AddUIVertexQuad` looks like it would render native quads. But the function converts a quad to two triangles internally. So don't expect some kind of performance boost.
- `AddTriangle` is somewhat confusing. It assumes you have already used `AddVert` to add some vertices. `AddTriangle` then accepts indices for the index buffer.

Complex Geometry
===

If you want to generate complex geometry, the functions in the previous section probably have too high overhead. You can then turn to the following set of functions:

```
public int currentVertCount {get;}
public int currentIndexCount {get;}
public void FillMesh(Mesh mesh)
public void AddUIVertexStream(List<UIVertex> verts, List<int> indices)
public void AddUIVertexTriangleStream(List<UIVertex> verts)
public void GetUIVertexStream(List<UIVertex> stream)
```

Some notes here:
- `FillMesh` and `GetUIVertexStream` both give you the existing geometry. It's personal preference whether you use the `Mesh` or the `List<UIVertex>`. You can then use this data to do geometry effects for example.
- `AddUIVertexStream` is probably going to be your first choice for complex geometry. It accepts a list of vertices and a list of indices. Be careful though, the vertices get added to the existing list of vertices in `VertexHelper`. So make sure to offset the list of indices with `currentVertCount`. If you forget this, probably only one of the meshes will get drawn.
- `AddUIVertexTriangleStream` is also a bit weird. It accepts a list of triangles. This means the length must be multiple of 3, and you can't reuse vertices with the help of an index buffer.

Performance considerations
===

The "simple" API is indeed much simpler to use, but it's going to be slow for complex geometry. You can only add a single vertex or quad at a time.
Using the "complex" API in a naive way tends to generate a lot of garbage. First intuition is probably to make new lists for each mesh you want to generate. In the end they get copied around a bunch of times, generating a lot of garbage in the process.
So far I've managed to remove all garbage in my own code, but Unity still generates a lot by copying your buffers to internal buffers. The following code sample shows a way to prevent garbage generation as much as possible. The important bits are in `OnPopulateMesh`. The other functions are included for completeness.

```
public List<MultiLine> multilines = new List<MultiLine>();
public List<Line> lines = new List<Line>();
public List<Quad> quads = new List<Quad>();

//Note(Simon): These buffers are allocated once, and reused each frame.
private List<UIVertex> vertexBuffer = new List<UIVertex>();
private List<int> indexBuffer = new List<int>();

protected override void OnPopulateMesh(VertexHelper vh)
{
    vh.Clear();
    //Note(Simon): These buffers get passed around to all draw functions. They add their vertices and indices to the end.
    vertexBuffer.Clear();
    indexBuffer.Clear();

    foreach (var quad in quads)
    {
        DrawQuad(vertexBuffer, indexBuffer, quad);
    }

    foreach (var multiline in multilines)
    {
        DrawMultiLine(vertexBuffer, indexBuffer, multiline);
    }

    //Note(Simon): This is where all meshes get passed to the VertexHelper.
    vh.AddUIVertexStream(vertexBuffer, indexBuffer);
}

private static void DrawMultiLine(List<UIVertex> vertices, List<int> indices, MultiLine line)
{
    var verts = new UIVertex[line.positions.Length * 2];
    var inds = new int[line.positions.Length * 6];

    int startIndex = vertices.Count;

    for (int i = 0; i < line.positions.Length; i++)
    {
        verts[i * 2].position = new Vector2(line.positions[i].x, line.positions[i].y - (line.thickness / 2));
        verts[i * 2].color = line.color;

        verts[i * 2 + 1].position = new Vector2(line.positions[i].x, line.positions[i].y + (line.thickness / 2));
        verts[i * 2 + 1].color = line.color;
    }

    for (int i = 0; i < line.positions.Length - 1; i++)
    {
        inds[i * 6 + 0] = startIndex + i * 2;
        inds[i * 6 + 1] = startIndex + i * 2 + 1;
        inds[i * 6 + 2] = startIndex + i * 2 + 3;
        inds[i * 6 + 3] = startIndex + i * 2;
        inds[i * 6 + 4] = startIndex + i * 2 + 3;
        inds[i * 6 + 5] = startIndex + i * 2 + 2;
    }

    vertices.AddRange(verts);
    indices.AddRange(inds);
}

private static void DrawQuad(List<UIVertex> vertices, List<int> indices, Quad quad)
{
    var verts = new UIVertex[4];
    var inds = new int[6];

    int startIndex = vertices.Count;

    var x1 = new Vector2(quad.pos.x, quad.pos.y);
    var x2 = new Vector2(quad.pos.x, quad.size.y);
    var y1 = new Vector2(quad.size.x, quad.pos.y);
    var y2 = new Vector2(quad.size.x, quad.size.y);

    verts[0].position = x1;
    verts[1].position = x2;
    verts[2].position = y1;
    verts[3].position = y2;

    verts[0].color = verts[1].color = verts[2].color = verts[3].color = Color.white;

    inds[0] = startIndex;
    inds[1] = startIndex + 1;
    inds[2] = startIndex + 3;
    inds[3] = startIndex;
    inds[4] = startIndex + 3;
    inds[5] = startIndex + 2;

    vertices.AddRange(verts);
    indices.AddRange(inds);
}
```
