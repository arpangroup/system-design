# Ray Casting and Ray Tracing — Step-by-Step Guide With Mathematical Foundations and Simulation

> **Goal:** Learn ray casting and ray tracing from first principles — starting with the vector algebra that underpins both, through a classic 2D DDA ray caster (the Wolfenstein-3D technique), through the full geometry and physics of a recursive 3D ray tracer (sphere/plane/triangle intersection, Phong/Blinn-Phong shading, shadows, reflection, refraction, Fresnel), and finish with **live, interactive simulations of both** that you can run directly in a browser.
>
> **Watch this first — a clear visual introduction to ray casting:** [https://www.youtube.com/watch?v=TOEi6T2mtHo](https://www.youtube.com/watch?v=TOEi6T2mtHo)

---

# 1. What We Are Building

Two related but distinct things, built in order:

1. **A 2D ray caster** — the technique behind *Wolfenstein 3D* (1992): a grid-based world, one ray per screen column, and a clever trick (the DDA algorithm) that finds wall hits without checking every cell. Fast enough to run in real time on 1990s hardware, and fast enough to run in your browser today with zero optimization effort.
2. **A 3D ray tracer** — a physically-motivated renderer that fires a ray per pixel into a 3D scene, computes exact geometric intersections with spheres/planes/triangles, and shades each hit point using the same lighting mathematics that underlies most of computer graphics: Lambertian diffuse, Phong/Blinn-Phong specular, shadow rays, recursive reflection, and Snell's-law refraction.

```text
Ray Casting (2D, this guide's §18-§28)          Ray Tracing (3D, this guide's §29-§59)
                                                
  Player                                          Camera
    |  one ray per screen COLUMN                    |  one ray per PIXEL
    v                                                v
  2D grid of wall cells                          3D scene of spheres/planes/triangles
    |  DDA: step cell-by-cell, no real math          |  exact algebra: solve for t analytically
    v                                                v
  Wall hit -> distance -> wall-slice height       Hit point -> normal -> shade -> maybe
                                                   spawn MORE rays (shadow/reflect/refract)
```

Every section from here forward either derives a piece of the mathematics both techniques depend on, or uses that mathematics to build working code — by the end, §68's live simulation is a ray tracer whose every line of code you will have derived yourself, not copied from a black box.

---

# 2. Learning Objectives

By the end of this guide you should be able to:

- Derive, not just recall, the ray-sphere intersection formula, the reflection formula, and Snell's law in vector form.
- Explain exactly what the DDA algorithm is doing at each step, and why it never needs a square root or a floating-point wall-crossing search.
- Explain why ray casting looks distorted ("fisheye") without a specific correction, and derive that correction.
- Build a Lambertian + Phong/Blinn-Phong shading model from the dot product upward, understanding what each term physically represents.
- Explain why ray tracing is recursive, what a "depth limit" is protecting against, and how reflection and refraction rays are generated.
- Know exactly where naive ray tracing's cost comes from, and what a bounding volume hierarchy does about it.

---

# 3. Ray Casting vs Ray Tracing: What's the Difference?

This distinction confuses people constantly, including in professional contexts, so it's worth being precise before writing a single line of code:

| | Ray Casting | Ray Tracing |
|---|---|---|
| Rays per hit | One ray, one hit, done | One ray can spawn *more* rays (shadow, reflection, refraction) — recursive |
| Typical dimensionality | 2D world, rendered to look pseudo-3D (Wolfenstein 3D, Doom's engine family) | True 3D scene geometry |
| What a "hit" gives you | A distance (used to draw a flat-shaded vertical strip) | A distance, a surface normal, a material — enough to compute realistic shading, shadows, and reflections |
| Light transport modeled | None — walls are flat-shaded or texture-mapped by distance only | Local illumination at minimum (Phong-family models); global illumination in advanced variants (§65) |
| Real-time feasibility (no GPU) | Yes — this is precisely why it powered early-90s games | Historically no for anything but tiny/low-res scenes; modern GPUs with dedicated ray-tracing hardware changed this |

**The one-sentence version:** ray casting is a specific, cheap, 2D-world technique that fakes a 3D look with a single non-recursive ray per column; ray tracing is a general, recursive, fully-3D simulation of how light rays travel, bounce, and combine. Historically "ray casting" is also used as a *general* term for "any technique that fires rays to determine visibility" — which is why the first bounce of a ray tracer (finding the nearest surface a camera ray hits) is sometimes itself called "casting a ray." Both usages are correct; §18 and §29 make clear which sense is meant at each point in this guide.

---

# 4. Recommended Technology Stack

| Concern | Choice | Why |
|---|---|---|
| Core algorithm code (this document) | Java | Matches this guide's Java-based derivations and keeps the math visible in typed, explicit code — no operator-overloading magic hiding a vector operation. |
| Live simulations (the HTML companion, §28/§68) | Vanilla JavaScript + `<canvas>` | Runs directly in a browser with zero build step or dependency — you can open the HTML file and watch both algorithms execute. |
| Output for the offline ray tracer | The PPM image format (§58) | The simplest possible image format to write by hand — three ASCII numbers per pixel, no compression, no library needed. |
| Math library | None — hand-rolled `Vec3` | Seeing every dot product, cross product, and normalization explicitly is the entire point of this guide. |

---

# 5. Project Structure

```text
raytracing/
├── src/main/java/com/example/raytracing/
│   ├── math/
│   │   ├── Vec3.java
│   │   └── Ray.java
│   ├── raycasting2d/
│   │   ├── GridWorld.java
│   │   ├── DdaRayCaster.java
│   │   └── RayCastRenderer.java
│   ├── geometry/
│   │   ├── Hittable.java
│   │   ├── HitRecord.java
│   │   ├── Sphere.java
│   │   ├── Plane.java
│   │   └── Triangle.java
│   ├── shading/
│   │   ├── Material.java
│   │   ├── Light.java
│   │   └── PhongShader.java
│   ├── scene/
│   │   ├── Scene.java
│   │   ├── Camera.java
│   │   └── BvhNode.java
│   └── RayTracer.java
└── src/main/resources/
    └── raytracer_output.ppm
```

---

# 6. Vectors in 3D: Points, Directions, and Basic Operations

A 3D vector `(x, y, z)` plays **two different roles** in graphics, and confusing them is a classic source of bugs:

- As a **point**, it names a location in space.
- As a **direction**, it names a displacement — "this far in x, this far in y, this far in z" — with no fixed location.

A ray's origin is a point; a ray's direction is a direction. Subtracting two points gives a direction (the displacement from one to the other); adding a direction to a point gives another point. This guide uses one `Vec3` class for both, exactly as most real graphics code does, but keeping the *role* straight in your head at each step is what makes the rest of this guide's formulas make sense instead of feeling like arbitrary rules.

```java
// math/Vec3.java
public class Vec3 {
    public final double x, y, z;
    public Vec3(double x, double y, double z) { this.x = x; this.y = y; this.z = z; }
}
```

---

# 7. Vector Addition, Subtraction, and Scalar Multiplication

```java
public Vec3 add(Vec3 o) { return new Vec3(x + o.x, y + o.y, z + o.z); }
public Vec3 subtract(Vec3 o) { return new Vec3(x - o.x, y - o.y, z - o.z); }
public Vec3 scale(double t) { return new Vec3(x * t, y * t, z * t); }
```

Geometrically: `A + B` places `B`'s displacement starting at the tip of `A` (the parallelogram rule); `A - B` gives the displacement *from* `B` *to* `A` — this single fact is what makes "direction to the light" compute as `lightPosition.subtract(hitPoint)` and not the other way around, a sign error that silently produces backwards shading if you get it wrong (§39 revisits this exact trap).

---

# 8. The Dot Product: Definition, Formula, and Geometric Meaning

```java
public double dot(Vec3 o) { return x * o.x + y * o.y + z * o.z; }
```

Algebraically, that's the entire definition. Its geometric meaning is what makes it indispensable for graphics:

$$\vec{a} \cdot \vec{b} = |\vec{a}|\,|\vec{b}|\cos\theta$$

where $\theta$ is the angle between the two vectors. Three consequences fall out of this immediately:

- If both vectors are **unit length** (§12), the dot product **is** $\cos\theta$ — no trigonometric function call needed to know "how aligned" two directions are.
- The dot product is **positive** when the angle is less than 90° (vectors point "the same general way"), **zero** at exactly 90° (perpendicular), and **negative** beyond 90° (pointing away from each other).
- $\vec{a} \cdot \vec{a} = |\vec{a}|^2$ — the dot product of a vector with itself is its squared length, which is exactly how §31 avoids a square root when testing ray-sphere intersection.

---

# 9. Using the Dot Product for Angles and Projections

Two uses this guide relies on constantly:

**Testing "is this surface facing the light?"** — if $\vec{N}$ (surface normal) and $\vec{L}$ (direction to light) are both unit vectors, `N.dot(L)` is $\cos\theta$ between them. A negative value means the light is *behind* the surface relative to this normal — exactly the `max(0, N·L)` clamp in §39's Lambertian formula.

**Projecting one vector onto another** — the length of $\vec{a}$'s projection onto unit vector $\hat{b}$ is simply $\vec{a} \cdot \hat{b}$, and the projected vector itself is $(\vec{a} \cdot \hat{b})\,\hat{b}$. §45's reflection derivation is built entirely from this one idea: decompose a vector into a piece parallel to the normal (via projection) and a piece perpendicular to it, then flip only the parallel piece.

---

# 10. The Cross Product: Definition, Formula, and Geometric Meaning

```java
public Vec3 cross(Vec3 o) {
    return new Vec3(
        y * o.z - z * o.y,
        z * o.x - x * o.z,
        x * o.y - y * o.x
    );
}
```

Where the dot product answers "how aligned are these two vectors," the cross product answers "what direction is perpendicular to *both* of them." Its magnitude is $|\vec{a}||\vec{b}|\sin\theta$ (largest when the vectors are perpendicular, zero when parallel — the opposite behavior from the dot product), and its direction follows the right-hand rule: point your fingers along $\vec{a}$, curl them toward $\vec{b}$, and your thumb points along $\vec{a} \times \vec{b}$.

This is exactly how §36 computes a triangle's surface normal: two of its edges, crossed, give a vector perpendicular to the triangle's plane — which *is* the surface normal, up to normalization and a sign convention for "which side is outward."

---

# 11. Normal Vectors and Why They Matter for Shading

A **surface normal** is a unit vector perpendicular to a surface at a given point, pointing *away* from the solid the surface bounds. Every shading calculation in §37–§44 — how bright a point looks, which direction light reflects off it, whether it's in shadow — is computed relative to this one vector, because it's the only thing at a hit point that encodes "which way is this surface facing." Get the normal's direction or sign wrong, and every downstream lighting calculation silently inverts or corrupts, which is why §36 and §51 treat "compute the normal correctly, including its sign" as a first-class step, not an afterthought.

---

# 12. Vector Normalization (Unit Vectors)

```java
public double length() { return Math.sqrt(x * x + y * y + z * z); }
public Vec3 normalize() { double len = length(); return new Vec3(x / len, y / len, z / len); }
```

A **unit vector** has length exactly 1. Normalizing a direction throws away its magnitude and keeps only "which way" — essential because almost every formula in this guide (the dot-product-as-cosine trick in §8, the reflection formula in §45, Snell's law in §47) is only correct, as written, when its input vectors are unit length. A shading bug where highlights look wrong or shadows are subtly misplaced is very often, in practice, a forgotten `.normalize()` call.

---

# 13. Parametric Line Equations: The Foundation of a Ray

A line through a point with a given direction is written parametrically as a function of one real number $t$:

$$P(t) = O + t\,\vec{D}$$

At $t = 0$, $P(0) = O$ — the starting point. As $t$ increases, you move further along $\vec{D}$. Every value of $t$ produces exactly one point on the line; every point on the line corresponds to exactly one $t$. This single equation, restricted to $t \geq 0$ (a **ray**, not a full line — you can't travel "backward" from the origin), is the mathematical object both ray casting and ray tracing are named after.

---

# 14. The Ray Equation: P(t) = Origin + t · Direction

```java
// math/Ray.java
public class Ray {
    public final Vec3 origin;
    public final Vec3 direction; // convention used throughout this guide: always normalized

    public Ray(Vec3 origin, Vec3 direction) {
        this.origin = origin;
        this.direction = direction.normalize();
    }

    public Vec3 pointAt(double t) { return origin.add(direction.scale(t)); }
}
```

Keeping `direction` normalized by convention (rather than re-normalizing everywhere it's used) is a deliberate simplification: it means `t` in `pointAt(t)` is literally "distance traveled along the ray," which makes comparing two hit distances (§32's "choose the nearest root") a direct, meaningful comparison rather than a comparison of two different, un-comparable scales.

---

# 15. Coordinate Systems: World Space, Camera Space, Screen Space

| Space | What coordinates mean in it |
|---|---|
| **World space** | Where objects actually live — a sphere's center at `(0, 1, -5)` means exactly that, in the scene's own fixed frame of reference. |
| **Camera space** | Coordinates relative to the camera: the camera sits at the origin, looking down one axis (by convention, often $-z$). Simplifies visibility and projection math. |
| **Screen space** | 2D pixel coordinates on the final image — `(x, y)` where `x` runs `0..width` and `y` runs `0..height`. |

§17's camera-ray generation is precisely the process of turning a **screen-space** pixel coordinate into a **world-space** ray — the bridge between "which pixel am I computing" and "which 3D geometry could that pixel be looking at."

---

# 16. The Camera Model: Field of View, Aspect Ratio, and the Image Plane

A camera is defined by a position, a look direction, an "up" vector (to fix its orientation/roll), a **field of view** (how wide an angle the camera sees, in degrees), and the output image's **aspect ratio** (width ÷ height). Together these define a virtual rectangle in 3D space — the **image plane** — placed one unit in front of the camera, sized so that its horizontal extent corresponds exactly to the field of view:

$$\text{halfHeight} = \tan\left(\frac{\text{fov}}{2}\right), \quad \text{halfWidth} = \text{halfHeight} \times \text{aspectRatio}$$

Every pixel in the final image corresponds to exactly one point on this virtual rectangle — §17 turns that correspondence into code.

---

# 17. Generating a Ray for Each Pixel (Camera Ray Generation)

```java
// scene/Camera.java
public class Camera {
    private final Vec3 origin;
    private final Vec3 lowerLeftCorner;
    private final Vec3 horizontal, vertical;

    public Camera(Vec3 lookFrom, Vec3 lookAt, Vec3 up, double fovDegrees, double aspectRatio) {
        double theta = Math.toRadians(fovDegrees);
        double halfHeight = Math.tan(theta / 2);
        double halfWidth = aspectRatio * halfHeight;

        Vec3 w = lookFrom.subtract(lookAt).normalize();      // points BACKWARD, away from lookAt — camera convention
        Vec3 u = up.cross(w).normalize();                    // "right" direction (§10's cross product)
        Vec3 v = w.cross(u);                                 // recomputed "up", guaranteed perpendicular to both

        this.origin = lookFrom;
        this.horizontal = u.scale(2 * halfWidth);
        this.vertical = v.scale(2 * halfHeight);
        this.lowerLeftCorner = origin.subtract(horizontal.scale(0.5)).subtract(vertical.scale(0.5)).subtract(w);
    }

    /** s, t range 0..1 across the image — this IS the screen-space-to-world-space bridge from §15. */
    public Ray getRay(double s, double t) {
        Vec3 pointOnImagePlane = lowerLeftCorner.add(horizontal.scale(s)).add(vertical.scale(t));
        return new Ray(origin, pointOnImagePlane.subtract(origin));
    }
}
```

For every pixel `(px, py)` in a `width × height` image, `s = px / width` and `t = py / height` — one call to `getRay(s, t)` per pixel is the entire bridge from "which pixel" to "which ray to trace," and it's the very first line of §52's main render loop.

---

# 18. What Is Ray Casting, Concretely?

Ray casting, in the classic (Wolfenstein-3D) sense, renders a **2D grid world** — a top-down map where each cell is either empty or a wall — as if you were standing inside it and looking around in 3D. The trick: for each vertical strip (column) of the output image, cast **one ray** from the player's position, in a slightly different direction per column, find the nearest wall it hits, and draw that column as a vertical strip whose height is inversely proportional to the hit distance (closer walls look taller). No 3D geometry, no lighting model — just distance-to-height, repeated once per column. Watch the video linked on the first page for a visual walkthrough of exactly this idea before diving into the algorithm below.

---

# 19. The 2D Grid World Representation

```java
// raycasting2d/GridWorld.java
public class GridWorld {
    private final int[][] map; // 0 = empty, non-zero = a wall (the value can select a texture/color)
    private final int width, height;

    public GridWorld(int[][] map) {
        this.map = map;
        this.height = map.length;
        this.width = map[0].length;
    }

    public boolean isWall(int cellX, int cellY) {
        if (cellX < 0 || cellX >= width || cellY < 0 || cellY >= height) return true; // out of bounds = solid
        return map[cellY][cellX] != 0;
    }
}
```

A world this simple — a 2D array of integers — is the entire "level geometry" a ray caster needs. The player's position `(posX, posY)` is a floating-point coordinate *within* this grid (e.g. `(4.5, 4.5)` is the center of cell `(4, 4)`).

---

# 20. Casting a Single Ray Through a Grid (DDA Algorithm)

The naive approach — step the ray forward by a tiny fixed distance and check "am I inside a wall cell yet?" — works, but its accuracy and cost are both tied to how small that step is: too large and you can step clean through a thin wall, too small and you waste enormous amounts of computation per ray. The **Digital Differential Analyzer (DDA)** algorithm instead steps **exactly one grid line at a time** — always landing precisely on a cell boundary, in the correct order, with no wasted steps and no risk of skipping a cell — using only additions, no square roots, inside its loop.

---

# 21. The Digital Differential Analyzer (DDA) Algorithm, Derived Step by Step

The question DDA answers, cell by cell: "how far do I have to travel along this ray to cross the *next* vertical grid line, versus the next *horizontal* grid line — and which happens first?"

```text
Given a ray direction (rayDirX, rayDirY):

1. deltaDistX = how far you travel ALONG THE RAY to cross one full grid cell in X
             = |1 / rayDirX|   (if rayDirX == 0, this is infinite — the ray never crosses a vertical line)
   deltaDistY = |1 / rayDirY|, symmetrically

2. stepX = +1 if rayDirX > 0, else -1   (which direction, in grid cells, we're moving in X)
   stepY = +1 if rayDirY > 0, else -1

3. sideDistX = the ray-distance from the START to the FIRST vertical grid line it will cross
             = (mapX + 1 - posX) * deltaDistX   if stepX > 0
             = (posX - mapX) * deltaDistX       if stepX < 0
   sideDistY computed symmetrically for the first horizontal grid line

4. Loop:
   - If sideDistX < sideDistY: the ray reaches the next VERTICAL line first.
         Advance: mapX += stepX; sideDistX += deltaDistX; record "hit a vertical (EW-facing) wall side"
     Else: the ray reaches the next HORIZONTAL line first.
         Advance: mapY += stepY; sideDistY += deltaDistY; record "hit a horizontal (NS-facing) wall side"
   - If GridWorld.isWall(mapX, mapY): STOP. This is the hit cell.
   - Otherwise, repeat step 4.
```

Why `deltaDistX = |1 / rayDirX|`: the ray's direction vector `(rayDirX, rayDirY)` advances by `rayDirX` in the x-coordinate for every 1 unit of ray-distance traveled (assuming the direction vector is treated as "per unit ray-length"). To advance by exactly `1` in x (one full grid cell), you need `1 / rayDirX` units of ray-distance — the absolute value because "distance traveled" is never negative even when `rayDirX` is.

---

# 22. Detecting Wall Hits and Computing Distance

```java
// raycasting2d/DdaRayCaster.java
public class DdaRayCaster {
    public record HitResult(double perpWallDist, boolean hitVerticalSide) { }

    public HitResult cast(GridWorld world, double posX, double posY, double rayDirX, double rayDirY) {
        int mapX = (int) posX, mapY = (int) posY;
        double deltaDistX = (rayDirX == 0) ? Double.MAX_VALUE : Math.abs(1 / rayDirX);
        double deltaDistY = (rayDirY == 0) ? Double.MAX_VALUE : Math.abs(1 / rayDirY);

        int stepX = rayDirX < 0 ? -1 : 1;
        int stepY = rayDirY < 0 ? -1 : 1;
        double sideDistX = (rayDirX < 0) ? (posX - mapX) * deltaDistX : (mapX + 1.0 - posX) * deltaDistX;
        double sideDistY = (rayDirY < 0) ? (posY - mapY) * deltaDistY : (mapY + 1.0 - posY) * deltaDistY;

        boolean hitVerticalSide = false;
        while (true) {
            if (sideDistX < sideDistY) {
                sideDistX += deltaDistX;
                mapX += stepX;
                hitVerticalSide = true;
            } else {
                sideDistY += deltaDistY;
                mapY += stepY;
                hitVerticalSide = false;
            }
            if (world.isWall(mapX, mapY)) break;
        }

        // perpendicular distance, NOT Euclidean distance to the hit point — see §23 for why this matters
        double perpWallDist = hitVerticalSide
            ? (sideDistX - deltaDistX)
            : (sideDistY - deltaDistY);
        return new HitResult(perpWallDist, hitVerticalSide);
    }
}
```

`hitVerticalSide` is worth its keep beyond bookkeeping: real ray casters shade vertical-side (east/west-facing) walls slightly differently from horizontal-side (north/south-facing) walls — a cheap, non-physical trick that gives the illusion of directional lighting for free.

---

# 23. Fixing the Fisheye Effect (Perpendicular Distance Correction)

If you render wall height from the **raw Euclidean distance** the DDA loop finds (straight-line distance from the player to the hit point), the result looks warped — walls bow outward like a fisheye lens, worse toward the edges of the screen. The cause: rays toward the edge of the field of view travel *further* than rays down the center to reach an object at the same perpendicular distance from the player, simply because they're angled.

The fix is to project that distance back onto the direction the player is *facing* — the **perpendicular distance** — rather than using the true point-to-point distance:

$$\text{perpWallDist} = \text{trueDistance} \times \cos(\theta)$$

where $\theta$ is the angle between this ray and the player's forward direction. DDA computes this perpendicular distance directly, as a side effect of how `sideDistX`/`sideDistY` accumulate (the `sideDistX - deltaDistX` line in §22) — no explicit cosine ever needs to be computed, which is a large part of why this specific algorithm, not a naive distance-based one, became the standard technique.

---

# 24. Rendering a Vertical Wall Slice From a Single Ray

```java
// raycasting2d/RayCastRenderer.java
public void renderColumn(Graphics g, int screenX, int screenHeight, DdaRayCaster.HitResult hit) {
    int lineHeight = (int) (screenHeight / hit.perpWallDist()); // closer wall (small dist) -> taller strip
    int drawStart = Math.max(0, -lineHeight / 2 + screenHeight / 2);
    int drawEnd = Math.min(screenHeight - 1, lineHeight / 2 + screenHeight / 2);

    Color color = hit.hitVerticalSide() ? Color.DARK_GRAY : Color.GRAY; // cheap directional shading, §22
    g.setColor(color);
    g.drawLine(screenX, drawStart, screenX, drawEnd);
}
```

`screenHeight / perpWallDist` is the entire "3D illusion": a wall twice as far away renders at half the height, which is exactly how perspective projection behaves for an object of fixed real-world size — derived here from a simple inverse relationship rather than a full projection matrix, because a ray caster's world is 2D and doesn't need one.

---

# 25. Casting a Ray for Every Screen Column

```java
public void renderFrame(Graphics g, GridWorld world, double posX, double posY,
                         double dirX, double dirY, double planeX, double planeY, int screenWidth, int screenHeight) {
    DdaRayCaster caster = new DdaRayCaster();
    for (int x = 0; x < screenWidth; x++) {
        double cameraX = 2.0 * x / screenWidth - 1; // ranges from -1 (left edge) to +1 (right edge)
        double rayDirX = dirX + planeX * cameraX;   // dir +/- a fraction of the camera "plane" vector -> field of view
        double rayDirY = dirY + planeY * cameraX;

        DdaRayCaster.HitResult hit = caster.cast(world, posX, posY, rayDirX, rayDirY);
        renderColumn(g, x, screenHeight, hit);
    }
}
```

The `plane` vector (perpendicular to `dir`, scaled to control field of view — a narrower plane vector means a narrower FOV) plays the exact same role here that `horizontal`/`vertical` played in §17's 3D camera: it's what turns "which screen column" into "which ray direction," just in 2D instead of 3D.

---

# 26. Phase 1 — Implementing the 2D Ray Caster in Java

Putting §19, §22, and §25 together is the entire ray caster — there is no additional machinery. A minimal `main` that renders one static frame:

```java
public static void main(String[] args) {
    int[][] map = {
        {1,1,1,1,1,1,1,1},
        {1,0,0,0,0,0,0,1},
        {1,0,1,0,1,0,0,1},
        {1,0,0,0,0,0,0,1},
        {1,1,1,1,1,1,1,1},
    };
    GridWorld world = new GridWorld(map);
    RayCastRenderer renderer = new RayCastRenderer();
    // posX/posY = player position, dirX/dirY = facing direction, planeX/planeY = camera plane (FOV)
    renderer.renderFrame(graphics, world, 3.5, 2.5, -1, 0, 0, 0.66, 640, 480);
}
```

---

# 27. Adding Player Movement and Rotation

Movement is just updating `(posX, posY)` along `(dirX, dirY)`, with a wall check so the player can't walk through geometry the ray caster itself renders as solid:

```java
public void moveForward(GridWorld world, double speed) {
    double newX = posX + dirX * speed;
    double newY = posY + dirY * speed;
    if (!world.isWall((int) newX, (int) posY)) posX = newX; // check X and Y independently -> sliding along walls
    if (!world.isWall((int) posX, (int) newY)) posY = newY;
}

public void rotate(double angle) {
    double oldDirX = dirX;
    dirX = dirX * Math.cos(angle) - dirY * Math.sin(angle); // standard 2D rotation matrix
    dirY = oldDirX * Math.sin(angle) + dirY * Math.cos(angle);
    double oldPlaneX = planeX;
    planeX = planeX * Math.cos(angle) - planeY * Math.sin(angle); // the camera plane must rotate WITH the view direction
    planeY = oldPlaneX * Math.sin(angle) + planeY * Math.cos(angle);
}
```

Checking the X and Y movement **independently** (rather than rejecting the whole move if the diagonal target cell is a wall) is what produces the natural "slide along the wall" feel instead of a hard stop the instant you brush a wall at an angle — a small detail with an outsized effect on how the simulation *feels* to move around in.

---

# 28. The Live Ray Casting Simulation (What You'll See in the HTML Version)

The HTML companion to this document embeds a **real, interactive, running implementation** of everything in §18–§27 — the same DDA algorithm, the same perpendicular-distance fisheye correction, the same independent-axis collision sliding — written in vanilla JavaScript against an HTML `<canvas>`, controllable with the arrow keys. There is no simplification between the Java code above and the JavaScript that actually runs: every formula in §21–§23 appears in that script unchanged, just in a different language. If you're reading the Markdown version of this guide, open the HTML version to drive it yourself — the Java above and the live JS are two renderings of the exact same algorithm.

---

# 29. From 2D Grids to 3D Scenes: What Changes

Ray tracing keeps the ray equation (§14) and the per-pixel ray generation (§17) completely unchanged from what a 3D-aware ray caster would already need — what's genuinely new is **what the ray is tested against**. Instead of "is this grid cell a wall," a ray tracer asks "does this ray intersect this sphere / this plane / this triangle," and needs that answer as an **exact real number** ($t$, the ray equation's parameter) rather than a "yes/no, stepped one grid cell at a time" answer. §30–§36 derive that exact intersection math for the three most common primitives.

---

# 30. Implicit Surfaces and the Sphere Equation

A sphere of radius $r$ centered at $C$ is the set of all points $P$ satisfying:

$$|P - C|^2 = r^2$$

This is an **implicit** equation — it doesn't tell you how to *generate* points on the sphere, only how to *test* whether a given point is on it (or, with $<$/$>$, inside/outside it). That test-ability is exactly what makes it easy to combine with the ray equation: substitute $P(t)$ from §14 in for $P$, and solve for the values of $t$ that satisfy the equation.

---

# 31. Deriving the Ray-Sphere Intersection Formula (the Quadratic)

Substitute the ray equation $P(t) = O + t\vec{D}$ into the sphere equation, writing $\vec{oc} = O - C$ for brevity:

$$|O + t\vec{D} - C|^2 = r^2 \quad\Longrightarrow\quad |\vec{oc} + t\vec{D}|^2 = r^2$$

Expand the squared length using the dot product ($|\vec{v}|^2 = \vec{v}\cdot\vec{v}$, §8):

$$(\vec{oc} + t\vec{D})\cdot(\vec{oc} + t\vec{D}) = r^2$$
$$\vec{oc}\cdot\vec{oc} + 2t(\vec{oc}\cdot\vec{D}) + t^2(\vec{D}\cdot\vec{D}) = r^2$$

Rearranged into standard quadratic form $at^2 + bt + c = 0$:

$$a = \vec{D}\cdot\vec{D}, \qquad b = 2(\vec{oc}\cdot\vec{D}), \qquad c = \vec{oc}\cdot\vec{oc} - r^2$$

If $\vec{D}$ is normalized (§14's convention), $a = 1$ always — one less multiplication per test, which matters when this test runs millions of times per render.

---

# 32. Solving the Quadratic: Discriminant, Roots, and Choosing the Nearest Hit

The quadratic formula gives $t = \dfrac{-b \pm \sqrt{b^2 - 4ac}}{2a}$. The term under the square root, the **discriminant**, tells you everything about whether (and how) the ray hits the sphere *before* you even compute a root:

| Discriminant | Meaning |
|---|---|
| $< 0$ | No real roots — the ray misses the sphere entirely |
| $= 0$ | Exactly one root — the ray grazes the sphere tangentially |
| $> 0$ | Two roots — the ray enters and exits the sphere |

```java
// geometry/Sphere.java
public HitRecord hit(Ray ray, double tMin, double tMax) {
    Vec3 oc = ray.origin.subtract(center);
    double a = ray.direction.dot(ray.direction);
    double b = 2.0 * oc.dot(ray.direction);
    double c = oc.dot(oc) - radius * radius;
    double discriminant = b * b - 4 * a * c;
    if (discriminant < 0) return null; // no intersection

    double sqrtD = Math.sqrt(discriminant);
    double t = (-b - sqrtD) / (2 * a);   // try the NEARER root first
    if (t < tMin || t > tMax) {
        t = (-b + sqrtD) / (2 * a);      // nearer root was behind us or out of range — try the farther one
        if (t < tMin || t > tMax) return null;
    }
    Vec3 hitPoint = ray.pointAt(t);
    Vec3 outwardNormal = hitPoint.subtract(center).scale(1.0 / radius); // cheaper than a fresh normalize() — see §36
    return new HitRecord(t, hitPoint, outwardNormal);
}
```

Trying the **nearer** root ($-b - \sqrt{\text{disc}}$) first, and only falling back to the farther one, is what correctly handles a camera positioned *inside* a sphere (the near root is behind the camera, at negative $t$) without any special-case code — the `tMin`/`tMax` bounds check does all the work.

---

# 33. Ray-Plane Intersection, Derived From the Plane Equation

A plane is defined by a point $P_0$ on it and a normal $\vec{N}$ perpendicular to it. A point $P$ lies on the plane exactly when the vector from $P_0$ to $P$ is perpendicular to $\vec{N}$ — i.e. their dot product is zero:

$$\vec{N} \cdot (P - P_0) = 0$$

Substituting the ray equation:

$$\vec{N} \cdot (O + t\vec{D} - P_0) = 0 \;\Longrightarrow\; \vec{N}\cdot(O - P_0) + t(\vec{N}\cdot\vec{D}) = 0$$

$$t = \frac{\vec{N}\cdot(P_0 - O)}{\vec{N}\cdot\vec{D}}$$

```java
// geometry/Plane.java
public HitRecord hit(Ray ray, double tMin, double tMax) {
    double denom = normal.dot(ray.direction);
    if (Math.abs(denom) < 1e-6) return null; // ray is (near) parallel to the plane — N·D ≈ 0, division would blow up
    double t = normal.dot(point.subtract(ray.origin)) / denom;
    if (t < tMin || t > tMax) return null;
    return new HitRecord(t, ray.pointAt(t), normal);
}
```

Unlike a sphere, a plane is infinite and unbounded — this `hit` method will report a hit anywhere on that infinite plane, which is exactly right for a "floor" but means a *finite* flat surface (a rectangle, a disc) needs an additional bounds check on the hit point after this formula finds $t$.

---

# 34. Ray-Triangle Intersection: The Möller–Trumbore Algorithm, Derived

A triangle with vertices $V_0, V_1, V_2$ can be described by two edge vectors $\vec{e_1} = V_1 - V_0$ and $\vec{e_2} = V_2 - V_0$: every point on the triangle's plane is $V_0 + u\vec{e_1} + v\vec{e_2}$ for some scalars $u, v$ (with $u, v \geq 0$ and $u+v \leq 1$ restricting you to *inside* the triangle, not just its plane — see §35). Setting this equal to the ray equation gives one vector equation in three unknowns ($t$, $u$, $v$):

$$O + t\vec{D} = V_0 + u\vec{e_1} + v\vec{e_2}$$

Rearranged into a linear system $t\vec{D} - u\vec{e_1} - v\vec{e_2} = O - V_0$, this is a standard 3-equations-in-3-unknowns linear system, solvable by **Cramer's rule** — and Cramer's rule for a 3×3 system is expressible entirely in terms of dot and cross products, which is exactly what the Möller–Trumbore algorithm's famously compact code is doing:

```java
// geometry/Triangle.java
public HitRecord hit(Ray ray, double tMin, double tMax) {
    Vec3 e1 = v1.subtract(v0);
    Vec3 e2 = v2.subtract(v0);
    Vec3 h = ray.direction.cross(e2);
    double a = e1.dot(h);
    if (Math.abs(a) < 1e-8) return null; // ray is parallel to the triangle's plane

    double f = 1.0 / a;
    Vec3 s = ray.origin.subtract(v0);
    double u = f * s.dot(h);
    if (u < 0 || u > 1) return null; // outside the triangle along the e1 direction

    Vec3 q = s.cross(e1);
    double v = f * ray.direction.dot(q);
    if (v < 0 || u + v > 1) return null; // outside the triangle along the e2 direction, or past the far edge

    double t = f * e2.dot(q);
    if (t < tMin || t > tMax) return null;

    Vec3 normal = e1.cross(e2).normalize(); // §10 and §36 — perpendicular to both edges
    return new HitRecord(t, ray.pointAt(t), normal);
}
```

Each rejected case above (`a` near zero, `u` out of `[0,1]`, `v` out of range, `u+v > 1`) corresponds to a distinct geometric "definitely no hit" fact, checked in the cheapest-first order — this is why Möller–Trumbore is the standard triangle-intersection routine in essentially every ray tracer and GPU rasterizer built since its 1997 publication.

---

# 35. Barycentric Coordinates and Why They Matter for Interpolation

The $(u, v)$ pair §34 solves for are two of a triangle's three **barycentric coordinates** — weights $(w_0, w_1, w_2) = (1 - u - v,\ u,\ v)$ that sum to 1 and describe a hit point as a weighted average of the triangle's three vertices: $P = w_0 V_0 + w_1 V_1 + w_2 V_2$. Their real power is that **any per-vertex quantity** — not just position, but a normal, a texture coordinate, a color — can be interpolated across the triangle's surface using these same three weights, which is exactly how smooth-shaded (as opposed to flat-shaded) triangle meshes compute a continuously-varying normal at every point on a triangle from just three stored vertex normals.

---

# 36. Computing Surface Normals at a Hit Point

| Primitive | Normal at a hit point |
|---|---|
| Sphere | $(P_{\text{hit}} - C) / r$ — the direction from the center outward, already unit length if divided by the true radius (§32) |
| Plane | The plane's defining normal, unchanged everywhere — a plane is flat by definition |
| Triangle (flat-shaded) | $\vec{e_1} \times \vec{e_2}$, normalized (§10, §34) — constant across the whole triangle |
| Triangle (smooth-shaded) | $w_0 N_0 + w_1 N_1 + w_2 N_2$ using §35's barycentric weights and the three vertices' *stored* normals, then re-normalized |

One subtlety worth internalizing now, because §43 and §46 both depend on it: a computed normal always points in a *consistent* geometric direction, but shading math (§39 onward) needs it to point toward the side the ray arrived *from*. If a ray can hit a surface from either side (the inside of a hollow sphere, a double-sided triangle), the normal must be flipped — `if (ray.direction.dot(outwardNormal) > 0) normal = normal.scale(-1);` — before it's handed to any shading calculation, or the lighting math silently produces nonsense on back-facing hits.

---

# 37. The Rendering Equation (Conceptual Introduction)

Every physically-motivated renderer, from this guide's simple Phong model to a full path tracer, is an approximation of one equation — the **rendering equation** (Kajiya, 1986), which says, informally: *the light leaving a point in a given direction equals the light the point emits, plus the light arriving from every other direction, weighted by how much the surface reflects light from that incoming direction into this outgoing one, integrated over the entire hemisphere above the surface.*

$$L_o(p, \omega_o) = L_e(p, \omega_o) + \int_{\Omega} f_r(p, \omega_i, \omega_o)\, L_i(p, \omega_i)\, (\vec{N}\cdot\omega_i)\, d\omega_i$$

A full path tracer (§65) attempts to estimate this integral directly via random sampling. This guide's **local illumination** model (§38–§44) is a drastic, deliberate simplification of it: instead of integrating over every incoming direction, it considers light from only a handful of *explicit* point lights, and ignores indirect light bouncing off other surfaces entirely (except for the one explicit reflection/refraction ray of §45–§50). Naming the full equation here is what makes it clear, later, exactly which physical effects this guide's model does and doesn't capture — and why.

---

# 38. Local Illumination: Ambient, Diffuse, and Specular Components

The classic Phong reflection model splits the light at a point into three additive terms:

$$I = I_{\text{ambient}} + I_{\text{diffuse}} + I_{\text{specular}}$$

| Term | Physical intuition | Depends on |
|---|---|---|
| **Ambient** | A crude stand-in for all the indirect light this simplified model doesn't otherwise compute — prevents unlit surfaces from being pure black | Nothing but a constant — no geometry, no light position |
| **Diffuse** (§39) | Light scattered equally in all directions by a rough surface (matte paint, chalk) | The angle between the surface normal and the direction to the light |
| **Specular** (§40–§41) | The bright highlight from a light source reflecting almost-directly off a shiny surface | The angle between the *reflected* light direction and the direction to the *viewer* |

---

# 39. Deriving Lambertian (Diffuse) Shading From the Dot Product

**Lambert's cosine law**, observed experimentally in 1760 and derivable from basic radiometry, states that the apparent brightness of a perfectly diffuse (matte) surface is proportional to the cosine of the angle between the surface normal and the direction to the light source — *not* to the viewer's position at all, which is why a matte surface looks equally bright from any viewing angle.

$$I_{\text{diffuse}} = k_d \, I_{\text{light}} \, \max(0,\ \vec{N} \cdot \vec{L})$$

where $\vec{N}$ is the (unit) surface normal, $\vec{L}$ is the (unit) direction **from the hit point toward the light** (§7's subtraction order matters here — `light.position.subtract(hitPoint)`, then normalized), and $k_d$ is the surface's diffuse reflectance (its "matte color"). The `max(0, ...)` clamp (§9) is what correctly darkens a surface facing away from the light to exactly zero, rather than a nonsensical negative brightness.

```java
double diffuseIntensity = Math.max(0, normal.dot(lightDir)) * light.intensity;
Vec3 diffuseColor = material.diffuseColor.scale(diffuseIntensity);
```

---

# 40. Deriving Phong Specular Shading

A shiny highlight is bright only when the viewer is looking almost exactly along the direction the light *would* perfectly mirror-reflect off the surface. Phong's model computes that mirror-reflection direction $\vec{R}$ (§45 derives this formula in full) and compares it to the direction toward the viewer $\vec{V}$:

$$I_{\text{specular}} = k_s \, I_{\text{light}} \, \max(0,\ \vec{R}\cdot\vec{V})^{n}$$

The exponent $n$ — the **shininess** — controls how tight the highlight is: a low $n$ (say, 8) spreads a soft, broad highlight; a high $n$ (say, 200) produces a small, sharp, mirror-like glint. Raising a value already in $[0,1]$ to a higher power pushes it toward zero faster, which is precisely why a larger exponent narrows the highlight — a purely mathematical consequence of exponentiation, not a separate physical rule.

---

# 41. Blinn-Phong: A Cheaper, More Physically Plausible Approximation

Computing the true reflection vector $\vec{R}$ per pixel, per light, is one extra formula's worth of work (§45) that Blinn-Phong avoids by introducing the **halfway vector** — the direction exactly between the light and the viewer:

$$\vec{H} = \text{normalize}(\vec{L} + \vec{V}), \qquad I_{\text{specular}} = k_s\, I_{\text{light}}\, \max(0,\ \vec{N}\cdot\vec{H})^{n'}$$

$\vec{N}\cdot\vec{H}$ is large exactly when $\vec{N}$ is "between" $\vec{L}$ and $\vec{V}$ in the same sense that $\vec{R}\cdot\vec{V}$ is large when $\vec{R}$ points at the viewer — the two formulas produce visually similar highlights (with a different, empirically-adjusted shininess exponent $n'$ needed to match brightness falloff), but Blinn-Phong also happens to avoid a specific artifact original Phong has at grazing angles, which is why it's the more common choice in real-time engines to this day.

---

# 42. Putting It Together: The Full Phong Lighting Model

```java
// shading/PhongShader.java
public Vec3 shade(HitRecord hit, Material material, Light light, Vec3 viewDir) {
    Vec3 lightDir = light.position.subtract(hit.point).normalize();     // §39 — direction TO the light
    Vec3 halfway = lightDir.add(viewDir).normalize();                   // §41

    double diffuseTerm = Math.max(0, hit.normal.dot(lightDir));
    double specularTerm = Math.pow(Math.max(0, hit.normal.dot(halfway)), material.shininess);

    Vec3 ambient = material.color.scale(material.ambientStrength);
    Vec3 diffuse = material.color.scale(diffuseTerm * light.intensity);
    Vec3 specular = new Vec3(1, 1, 1).scale(specularTerm * light.intensity); // highlights are usually near-white, not the surface color

    return ambient.add(diffuse).add(specular);
}
```

Specular highlights being tinted toward white (rather than the surface's own color) is a deliberate, physically-motivated choice: for most non-metallic materials, the light bouncing straight off the surface without penetrating it at all retains the light's color, not the material's — this is the same underlying physics that makes a shiny red apple's highlight look white, not red.

---

# 43. Shadows: Casting a Shadow Ray Toward the Light

§42's shading assumes the light can actually "see" the hit point — if another object sits between them, the point should be in shadow. The test: fire a second ray, a **shadow ray**, from the hit point toward the light, and check whether it hits anything *before* reaching the light's distance.

```java
public boolean isInShadow(Scene scene, Vec3 hitPoint, Vec3 lightDir, double distanceToLight) {
    Vec3 shadowRayOrigin = hitPoint.add(lightDir.scale(1e-4)); // §44's "shadow acne" fix — see the note below
    Ray shadowRay = new Ray(shadowRayOrigin, lightDir);
    HitRecord blocker = scene.hitAnything(shadowRay, 0.001, distanceToLight);
    return blocker != null;
}
```

**Why nudge the shadow ray's origin slightly off the surface?** Due to ordinary floating-point rounding, a ray whose origin is computed as "exactly on the surface" can, due to tiny rounding error, register as intersecting that *same* surface again at $t \approx 0$ — a bug universally nicknamed **shadow acne**, appearing as dark speckling across lit surfaces. Nudging the origin a small distance along the normal (or, as above, along the shadow ray direction) pushes it just clear of the surface, avoiding the spurious self-intersection without introducing a visible gap.

---

# 44. Multiple Lights and Light Attenuation

Real scenes have more than one light — §42's `shade` is called once per light, and the results are summed:

```java
Vec3 totalColor = material.color.scale(material.ambientStrength); // ambient term added once, not per light
for (Light light : scene.getLights()) {
    Vec3 lightDir = light.position.subtract(hit.point).normalize();
    double distanceToLight = light.position.subtract(hit.point).length();
    if (isInShadow(scene, hit.point, lightDir, distanceToLight)) continue; // this light contributes nothing here
    double attenuation = 1.0 / (1.0 + 0.09 * distanceToLight + 0.032 * distanceToLight * distanceToLight);
    totalColor = totalColor.add(phongShader.shade(hit, material, light, viewDir).scale(attenuation));
}
```

**Attenuation** — light weakening with distance — follows an inverse-square law physically ($1/d^2$), but a pure inverse-square term blows up unrealistically as $d \to 0$; the quadratic-plus-linear-plus-constant formula above (a common real-time-graphics approximation) tames that behavior while still falling off believably at typical scene distances.

---

# 45. The Law of Reflection, Derived From the Normal Vector

Decompose an incoming direction $\vec{D}$ (pointing *toward* the surface) into two components relative to the normal $\vec{N}$: a part parallel to $\vec{N}$, and a part perpendicular to it (lying in the surface's plane). Using §9's projection formula, the parallel part is $(\vec{D}\cdot\vec{N})\vec{N}$, and the perpendicular part is whatever's left: $\vec{D} - (\vec{D}\cdot\vec{N})\vec{N}$.

The law of reflection says the perpendicular component is **unchanged** by a mirror bounce, while the parallel component **flips sign** — physically, the reflected ray leaves at the same angle it arrived, on the other side of the normal, in the same plane. Flipping only the parallel part and re-adding the unperturbed perpendicular part:

$$\vec{R} = \big(\vec{D} - (\vec{D}\cdot\vec{N})\vec{N}\big) - (\vec{D}\cdot\vec{N})\vec{N} = \vec{D} - 2(\vec{D}\cdot\vec{N})\vec{N}$$

---

# 46. Implementing Reflection Rays

```java
public static Vec3 reflect(Vec3 incident, Vec3 normal) {
    return incident.subtract(normal.scale(2 * incident.dot(normal)));
}
```

```java
Vec3 reflectedDir = reflect(ray.direction, hit.normal);
Ray reflectedRay = new Ray(hit.point.add(hit.normal.scale(1e-4)), reflectedDir); // same "nudge off the surface" fix as §43
Vec3 reflectedColor = traceRay(scene, reflectedRay, depth - 1); // §49 — recursion, with a decrementing depth budget
finalColor = finalColor.add(reflectedColor.scale(material.reflectivity));
```

`material.reflectivity` (a value in $[0, 1]$) blends between the surface's own shaded color and whatever the reflected ray sees — `0` is a fully matte, non-reflective material; `1` is a perfect mirror; intermediate values (a shiny-but-not-mirror-like plastic, for instance) blend the two.

---

# 47. Snell's Law and Refraction, Derived

When light passes from a medium with refractive index $n_1$ (e.g. air, $n_1 \approx 1.0$) into one with index $n_2$ (e.g. glass, $n_2 \approx 1.5$), it bends according to **Snell's law**:

$$n_1 \sin\theta_1 = n_2 \sin\theta_2$$

where $\theta_1$ is the angle of incidence (from the normal) and $\theta_2$ is the angle of refraction. Deriving the *vector* form (not just the scalar angle relationship) takes a bit more geometry: decompose the incident direction into components parallel and perpendicular to the normal (same idea as §45), scale the perpendicular component by $\eta = n_1/n_2$ (Snell's law directly relates the *sines*, which are proportional to the perpendicular components), and solve for the parallel component using the fact that the refracted direction must remain unit length:

```java
public static Vec3 refract(Vec3 incident, Vec3 normal, double eta) {
    double cosThetaI = -normal.dot(incident);
    double sin2ThetaT = eta * eta * (1.0 - cosThetaI * cosThetaI);
    if (sin2ThetaT > 1.0) return null; // total internal reflection — see §50, no refraction is possible
    double cosThetaT = Math.sqrt(1.0 - sin2ThetaT);
    return incident.scale(eta).add(normal.scale(eta * cosThetaI - cosThetaT));
}
```

`eta = n1 / n2` is the ratio of refractive indices for the two media the ray is crossing — computed as $1/1.5$ when entering glass from air, and its reciprocal, $1.5/1$, when exiting glass back into air (tracking "which medium am I currently inside" is a real bookkeeping responsibility a recursive ray tracer with transparent objects must handle explicitly, easy to get backwards).

---

# 48. The Fresnel Effect: Why Glass Reflects and Refracts at Once

Look at a window at a shallow, grazing angle and it becomes far more reflective than looking straight through it — this is the **Fresnel effect**: the fraction of light reflected (versus refracted/transmitted) at a boundary between two media depends on the viewing angle, always increasing toward 100% reflection as the angle becomes more grazing. The full physical formula (the Fresnel equations) is involved; **Schlick's approximation** captures the same behavior cheaply:

$$R_0 = \left(\frac{n_1 - n_2}{n_1 + n_2}\right)^2, \qquad R(\theta) = R_0 + (1 - R_0)(1 - \cos\theta)^5$$

```java
public static double schlickReflectance(double cosine, double refractiveIndexRatio) {
    double r0 = (1 - refractiveIndexRatio) / (1 + refractiveIndexRatio);
    r0 = r0 * r0;
    return r0 + (1 - r0) * Math.pow(1 - cosine, 5);
}
```

$R(\theta)$ is used directly as a blend weight: a glass sphere's final color at a given hit point is `reflectedColor * R(θ) + refractedColor * (1 - R(θ))` — a physically-motivated mix rather than an artist-chosen constant, and the reason a well-implemented glass material looks convincingly like *glass* instead of like a semi-transparent colored filter.

---

# 49. Recursive Ray Tracing: Bouncing Rays and Depth Limiting

A perfectly reflective sphere facing a mirror could, in principle, bounce a ray back and forth forever — every recursive ray tracer therefore takes a **depth** parameter, decremented on every bounce, that forces the recursion to stop and return a default color (usually black, or the scene's background/ambient color) once exhausted:

```java
public Vec3 traceRay(Scene scene, Ray ray, int depth) {
    if (depth <= 0) return Vec3.BLACK; // depth budget exhausted — stop recursing, contribute nothing further
    HitRecord hit = scene.hitNearest(ray, 0.001, Double.MAX_VALUE);
    if (hit == null) return scene.backgroundColor(ray);

    Vec3 localColor = computeLocalShading(scene, hit); // §42-§44 — ambient + diffuse + specular + shadows
    Vec3 reflectedColor = Vec3.BLACK, refractedColor = Vec3.BLACK;

    if (hit.material.reflectivity > 0) {
        Ray reflectedRay = new Ray(hit.point.add(hit.normal.scale(1e-4)), reflect(ray.direction, hit.normal));
        reflectedColor = traceRay(scene, reflectedRay, depth - 1); // recursive call, ONE LESS depth remaining
    }
    if (hit.material.transparency > 0) {
        Vec3 refractDir = refract(ray.direction, hit.normal, hit.enteringMaterial ? 1.0 / hit.material.refractiveIndex : hit.material.refractiveIndex);
        if (refractDir != null) {
            Ray refractedRay = new Ray(hit.point.subtract(hit.normal.scale(1e-4)), refractDir); // nudge INTO the surface, not away
            refractedColor = traceRay(scene, refractedRay, depth - 1);
        } else {
            reflectedColor = traceRay(scene, new Ray(hit.point.add(hit.normal.scale(1e-4)), reflect(ray.direction, hit.normal)), depth - 1); // §50
        }
    }
    return localColor.scale(1 - hit.material.reflectivity - hit.material.transparency)
        .add(reflectedColor.scale(hit.material.reflectivity))
        .add(refractedColor.scale(hit.material.transparency));
}
```

The `depth` parameter is doing exactly the same conceptual job as a base case in any other recursive algorithm — without it, `traceRay` has no proof of termination, and a scene with two facing mirrors would recurse until the call stack overflows.

---

# 50. Total Internal Reflection

§47's `refract` function returns `null` whenever `sin2ThetaT > 1.0` — a real physical phenomenon, not a numerical edge case to paper over: when light tries to exit a *denser* medium into a *less dense* one (glass to air, not air to glass) at a sufficiently shallow angle, Snell's law has **no solution** — $\sin\theta_2$ would need to exceed 1, which is impossible for a real angle. Physically, 100% of the light reflects instead of refracting at all — this is exactly how fiber-optic cables work, and exactly why looking up from underwater at a shallow angle shows a mirror-like reflection of the pool floor instead of a view of the sky. §49's fallback to a pure reflection ray in that branch is not a hack — it's the physically correct behavior for this specific geometric case.

---

# 51. Phase 2 — Designing the Scene Graph (Spheres, Planes, Materials, Lights)

```java
// geometry/Hittable.java — the one interface every intersectable object implements
public interface Hittable {
    HitRecord hit(Ray ray, double tMin, double tMax);
}

// geometry/HitRecord.java
public class HitRecord {
    public final double t;
    public final Vec3 point;
    public final Vec3 normal;
    public final Material material;
    public HitRecord(double t, Vec3 point, Vec3 normal, Material material) {
        this.t = t; this.point = point; this.normal = normal; this.material = material;
    }
}

// shading/Material.java
public class Material {
    public Vec3 color;
    public double ambientStrength, shininess, reflectivity, transparency, refractiveIndex;
}

// shading/Light.java
public class Light {
    public Vec3 position;
    public double intensity;
}

// scene/Scene.java
public class Scene {
    private final List<Hittable> objects = new ArrayList<>();
    private final List<Light> lights = new ArrayList<>();

    public HitRecord hitNearest(Ray ray, double tMin, double tMax) {
        HitRecord closest = null;
        double closestT = tMax;
        for (Hittable object : objects) {
            HitRecord hit = object.hit(ray, tMin, closestT);
            if (hit != null) { closest = hit; closestT = hit.t; }
        }
        return closest;
    }
}
```

Every primitive (`Sphere`, `Plane`, `Triangle` from §32–§34) implements the same `Hittable` interface — `Scene.hitNearest` never needs to know which kind of geometry it's testing against, only that it can ask "does this ray hit you, and if so, how far away, and closer than the closest hit so far?"

---

# 52. Phase 3 — The Core Ray Tracing Loop

```java
// RayTracer.java
public BufferedImage render(Scene scene, Camera camera, int width, int height, int maxDepth) {
    BufferedImage image = new BufferedImage(width, height, BufferedImage.TYPE_INT_RGB);
    for (int py = 0; py < height; py++) {
        for (int px = 0; px < width; px++) {
            double s = (double) px / (width - 1);
            double t = (double) (height - 1 - py) / (height - 1); // flip Y — image rows go top-to-bottom, world Y is up
            Ray ray = camera.getRay(s, t);                        // §17
            Vec3 color = traceRay(scene, ray, maxDepth);           // §49
            image.setRGB(px, py, toRgbInt(color));
        }
    }
    return image;
}
```

This double loop — one call to `getRay` and one call to `traceRay` per pixel, nothing else — is the entire "outer shell" of a ray tracer. Every section from §29 onward exists to make one of those two function calls meaningful.

---

# 53. Phase 4 — Implementing Diffuse and Specular Shading in Code

Already fully derived and coded in §39–§42 (`PhongShader.shade`) — this phase is purely the integration step: `computeLocalShading` (referenced in §49) loops over every light in the scene and calls `PhongShader.shade` once per light, summing the results, exactly as shown in §44.

---

# 54. Phase 5 — Implementing Shadows in Code

Also already fully derived in §43 (`isInShadow`) — the integration point is `computeLocalShading` skipping a light's contribution entirely when `isInShadow` returns true, as shown in §44's loop (`if (isInShadow(...)) continue;`).

---

# 55. Phase 6 — Implementing Reflections in Code

Fully derived in §45–§46 and wired into the main recursive loop in §49 — nothing further to add here; this phase exists in the roadmap only to mark it as a distinct, independently-testable milestone (§67's testing strategy treats it as such).

---

# 56. Phase 7 — Implementing Refraction in Code

Fully derived in §47–§48 and wired into §49's recursive loop, including the Fresnel-weighted blend between reflection and refraction and the total-internal-reflection fallback (§50). Like §55, this phase is complete by construction of the earlier sections — called out here as its own milestone because it is, in practice, the single hardest part of a ray tracer to get bug-free (sign errors in `eta`, forgetting to nudge the refracted ray *into* rather than away from the surface, and mismatched "which medium am I in" bookkeeping are all extremely common).

---

# 57. Phase 8 — Anti-Aliasing via Supersampling

A single ray per pixel produces jagged, "stair-stepped" edges wherever an object's silhouette doesn't align with the pixel grid — because each pixel's color is an all-or-nothing sample of whatever the ray happened to hit. **Supersampling** fires *several* rays per pixel, at slightly jittered sub-pixel positions, and averages the results:

```java
public Vec3 samplePixelAntiAliased(Scene scene, Camera camera, int px, int py, int width, int height, int samplesPerPixel, int maxDepth) {
    Vec3 colorSum = Vec3.BLACK;
    for (int sample = 0; sample < samplesPerPixel; sample++) {
        double jitterX = Math.random(); // a random offset WITHIN this pixel's footprint
        double jitterY = Math.random();
        double s = (px + jitterX) / (width - 1);
        double t = (height - 1 - (py + jitterY)) / (double) (height - 1);
        colorSum = colorSum.add(traceRay(scene, camera.getRay(s, t), maxDepth));
    }
    return colorSum.scale(1.0 / samplesPerPixel);
}
```

Averaging several randomly-jittered samples per pixel means a pixel straddling an edge gets a color *blended* between the two sides in proportion to how much of the pixel's area each side actually covers — smoothing the jaggedness into a gradient the eye reads as a clean edge, at the direct cost of `samplesPerPixel` times more ray-tracing work per pixel.

---

# 58. Phase 9 — Writing the Output Image (PPM Format)

The **PPM** (Portable Pixmap) format is about as simple as an image format can be: a short text header, then three ASCII numbers (red, green, blue, each `0-255`) per pixel, row by row:

```java
public void writePpm(BufferedImage image, Path outputPath) throws IOException {
    try (PrintWriter writer = new PrintWriter(Files.newBufferedWriter(outputPath))) {
        int width = image.getWidth(), height = image.getHeight();
        writer.println("P3");                 // "P3" = ASCII PPM, as opposed to "P6" (binary)
        writer.println(width + " " + height);
        writer.println("255");                 // maximum value per color channel
        for (int y = 0; y < height; y++) {
            for (int x = 0; x < width; x++) {
                int rgb = image.getRGB(x, y);
                writer.println(((rgb >> 16) & 0xFF) + " " + ((rgb >> 8) & 0xFF) + " " + (rgb & 0xFF));
            }
        }
    }
}
```

No library, no compression, no binary parsing — just three numbers per pixel — which is exactly why PPM is the traditional first output format for a hand-built ray tracer: it proves the renderer works with the absolute minimum of code standing between "computed colors" and "a file you can open and look at" (most image viewers, and tools like GIMP or ImageMagick, read `.ppm` natively).

---

# 59. Full Worked Example: Rendering Three Spheres and a Floor Plane

```java
public static void main(String[] args) throws IOException {
    Scene scene = new Scene();
    scene.add(new Sphere(new Vec3(0, 0, -1), 0.5, matteRed));
    scene.add(new Sphere(new Vec3(1.2, 0, -1.5), 0.5, shinyBlue));      // higher shininess, some reflectivity
    scene.add(new Sphere(new Vec3(-1.2, 0, -1.5), 0.5, glassMaterial)); // transparency + refractiveIndex = 1.5
    scene.add(new Plane(new Vec3(0, -0.5, 0), new Vec3(0, 1, 0), checkerFloor));
    scene.addLight(new Light(new Vec3(2, 2, 1), 1.0));

    Camera camera = new Camera(new Vec3(0, 0.5, 1), new Vec3(0, 0, -1), new Vec3(0, 1, 0), 60, 16.0 / 9.0);
    RayTracer tracer = new RayTracer();
    BufferedImage image = tracer.render(scene, camera, 800, 450, /* maxDepth */ 5);
    tracer.writePpm(image, Path.of("raytracer_output.ppm"));
}
```

Tracing what happens for one pixel that lands on the glass sphere: `traceRay` finds the nearest hit (§32's quadratic), computes local Phong shading including a shadow-ray check against the floor and other spheres (§43–§44), computes both a reflected ray (§46) and — since `transparency > 0` — a refracted ray via Snell's law (§47), blends all three using the material's reflectivity/transparency weights (ideally Fresnel-weighted per §48), and recurses up to 4 more times (`maxDepth - 1`) for whatever those secondary rays hit, before finally returning one RGB color for that single pixel. Every section from §6 to §58 contributes to that one pixel's final color.

---

# 60. Why Naive Ray Tracing Is Slow: The Intersection Test Bottleneck

`Scene.hitNearest` (§51), as written, tests **every** object in the scene against **every** ray — for $N$ objects and $W \times H$ pixels at 1 ray/pixel, that's $O(N \times W \times H)$ intersection tests before even counting §57's supersampling multiplier or §49's recursive secondary rays. A scene with a few spheres is fine; a scene with a million triangles (an imported 3D model) is not — the vast majority of those tests are against objects nowhere near the ray's actual path, wasted work in an amount that grows linearly with scene complexity for every single ray.

---

# 61. Acceleration Structures: Bounding Volume Hierarchies (BVH)

A **Bounding Volume Hierarchy** wraps groups of objects in progressively larger bounding boxes, arranged as a binary tree — testing a ray against the *root* box first, and only descending into a child box (and eventually the real geometry) if the ray actually hits that box. A ray that misses a large region of the scene entirely is rejected with **one** cheap box test instead of testing every object inside that region individually.

```text
                    [Box containing EVERYTHING]
                    /                        \
        [Box: left half of scene]    [Box: right half of scene]
           /            \                /              \
      [Sphere A]    [Sphere B]      [Triangle 1]    [Box: 500 more triangles]
                                                          /          \
                                                   ... recursively subdivided ...
```

Ray-box intersection itself is cheap and simple (an **axis-aligned bounding box**, or AABB, test — three interval overlaps, one per axis, no square roots), which is exactly what makes the BVH's "reject early, cheaply" strategy pay for itself.

---

# 62. Building a BVH, Step by Step

```java
// scene/BvhNode.java
public class BvhNode implements Hittable {
    private final Hittable left, right;
    private final AABB boundingBox;

    public static BvhNode build(List<Hittable> objects) {
        if (objects.size() == 1) return new BvhNode(objects.get(0), objects.get(0));
        int axis = pickLongestAxis(objects);                          // split along whichever axis the objects spread out most
        objects.sort(Comparator.comparingDouble(o -> o.boundingBox().centroid(axis)));
        int mid = objects.size() / 2;
        Hittable leftSubtree = build(objects.subList(0, mid));         // recursively build each half
        Hittable rightSubtree = build(objects.subList(mid, objects.size()));
        return new BvhNode(leftSubtree, rightSubtree);
    }

    @Override
    public HitRecord hit(Ray ray, double tMin, double tMax) {
        if (!boundingBox.hit(ray, tMin, tMax)) return null;             // the entire optimization, in one line
        HitRecord leftHit = left.hit(ray, tMin, tMax);
        HitRecord rightHit = right.hit(ray, tMin, leftHit != null ? leftHit.t : tMax); // shrink the search range if left already found something closer
        return rightHit != null ? rightHit : leftHit;
    }
}
```

Building the tree by repeatedly splitting along the axis the objects spread out most, and recursing until each leaf holds one object, turns $O(N)$ per-ray intersection cost into $O(\log N)$ for a reasonably balanced tree — the same complexity win a balanced binary search tree gives over a linear scan through a sorted array, applied to 3D space instead of a 1D key.

---

# 63. Multithreading the Render Loop

Every pixel's color (§52) depends only on that pixel's own ray and the (read-only, once the scene is built) `Scene` — no pixel's computation depends on any other pixel's result, which makes ray tracing an unusually easy problem to parallelize:

```java
public BufferedImage renderParallel(Scene scene, Camera camera, int width, int height, int maxDepth) {
    BufferedImage image = new BufferedImage(width, height, BufferedImage.TYPE_INT_RGB);
    IntStream.range(0, height).parallel().forEach(py -> {  // each row is independent — safe to run across threads
        for (int px = 0; px < width; px++) {
            Vec3 color = traceRay(scene, camera.getRay((double) px / width, (double) py / height), maxDepth);
            synchronized (image) { image.setRGB(px, py, toRgbInt(color)); } // BufferedImage isn't thread-safe to write concurrently
        }
    });
    return image;
}
```

This is the same "embarrassingly parallel" property that made ray tracing an early, natural fit for GPU compute — every pixel's work is independent, so the only synchronization needed anywhere in this loop is around the shared output buffer itself, not around any part of the actual tracing logic.

---

# 64. Soft Shadows via Area Lights and Multiple Samples

§43's shadow rays produce perfectly sharp shadow edges — physically correct only for an infinitesimally small ("point") light source. Real lights have area (a light bulb, a window), which produces **soft shadows** — a gradual penumbra where the light source is partially, but not fully, occluded. Approximating this: treat a light as a small disc or sphere, and for each shading calculation, cast **several** shadow rays toward randomly-sampled points on that light's surface instead of one ray toward its center, averaging the resulting shadow/lit fraction:

```java
double litFraction = 0;
for (int i = 0; i < shadowSamples; i++) {
    Vec3 sampledLightPoint = light.position.add(randomPointOnDisc(light.radius));
    if (!isInShadow(scene, hit.point, sampledLightPoint.subtract(hit.point).normalize(), sampledLightPoint.subtract(hit.point).length())) {
        litFraction += 1.0 / shadowSamples;
    }
}
```

A point fully visible to the light gets `litFraction = 1` (fully lit), a point fully blocked gets `0` (fully shadowed), and a point at the shadow's edge — visible to *some* sampled points on the light but not others — gets a value in between, which is exactly the soft gradient a penumbra should have.

---

# 65. Global Illumination and Path Tracing: A Glimpse Beyond This Guide

Everything built through §64 is **local illumination with a fixed set of explicit secondary rays** (one reflection, one refraction, a handful of shadow rays) — it never accounts for light bouncing off *one ordinary diffuse surface onto another* (a red wall subtly tinting a nearby white object red, for instance), an effect called **indirect illumination** or **color bleeding**. §37's full rendering equation captures this in its integral over every incoming direction; **path tracing** approximates that integral via Monte Carlo sampling — firing many random rays per pixel, each bouncing multiple times off diffuse surfaces according to the surface's reflectance distribution, and averaging enormous numbers of samples to converge on a noise-free image. This is a substantially larger topic (importance sampling, Monte Carlo variance/convergence, denoising) than this guide's scope, but every piece of vector math, intersection code, and shading logic built here (§6–§59) is exactly what a path tracer is built from — the difference is *how many* rays get fired per pixel and how their contributions get combined, not a different geometric foundation.

---

# 66. Common Mistakes

- **Mistake 1 — Forgetting to normalize a ray direction (§14).** Silently breaks every formula in this guide that assumes unit-length inputs, most confusingly the quadratic in §31 (`a` is no longer 1) and the dot-product-as-cosine shortcut in §8.
- **Mistake 2 — Sign errors in "direction to the light" (§7, §39).** Computing `hitPoint.subtract(light.position)` instead of `light.position.subtract(hitPoint)` produces a direction pointing *away* from the light, silently darkening every surface that should be lit.
- **Mistake 3 — Shadow acne from skipping the origin nudge (§43).** A shadow ray that isn't offset slightly off the surface can spuriously re-intersect the very surface it started from, producing speckled dark noise on lit surfaces.
- **Mistake 4 — No recursion depth limit (§49).** A scene with two facing mirrors (or a glass object bouncing internally) recurses without bound, eventually overflowing the call stack.
- **Mistake 5 — Getting `eta` (the refractive index ratio) backwards or failing to track "which medium am I in" (§47, §56).** Produces refraction that bends the wrong way, or a glass object that looks correct entering but wrong exiting.
- **Mistake 6 — Choosing the farther root instead of the nearer one in ray-sphere intersection (§32).** Renders the back face of a sphere as if it were the front, or fails to correctly handle a camera positioned inside an object.
- **Mistake 7 — Treating DDA's `sideDistX`/`sideDistY` as the true hit distance instead of using the perpendicular distance (§23).** Reintroduces the fisheye distortion the entire point of the DDA bookkeeping was designed to avoid.
- **Mistake 8 — Testing every object against every ray with no acceleration structure (§60) on a scene with thousands of triangles.** Correct, but slow enough to make iteration on a real scene painfully slow — the first thing to fix once render times become the bottleneck.

---

# 67. Testing Strategy (Comparing Against Known Analytical Results)

| What to test | Known correct answer | Why it's a good test |
|---|---|---|
| Ray straight at a sphere's center | Hit distance = `distance(origin, center) - radius` exactly | A simple, hand-computable case that catches sign/formula errors in §31-§32 immediately |
| Ray tangent to a sphere | Discriminant exactly (or very nearly) zero | Exercises the boundary case between "hit" and "miss" |
| Ray parallel to a plane | No hit, ever, regardless of distance | Catches a missing or incorrect near-zero-denominator check in §33 |
| A point light directly above a flat diffuse surface, viewed from directly above | Diffuse term should equal `k_d * lightIntensity` exactly (`N·L = 1`) | A precise, hand-computable check on §39's formula |
| A fully-reflective sphere next to a fully-diffuse sphere of a known color | The reflective sphere's surface should show a recognizable, undistorted-in-principle mirror image of the diffuse one | Validates §45-§46's reflection direction is geometrically correct, not just "looks shiny" |
| A glass sphere at normal incidence (looking straight through it) | Should show the scene behind it with minimal distortion and near-zero reflection (Fresnel at 0° is close to `R0`, small for glass) | Validates §47-§48 together |
| DDA against a single-cell-thick wall at a shallow grazing angle | The wall must never be "skipped" (a naive fixed-step raymarcher can miss it; DDA, correctly implemented, cannot) | The core correctness guarantee DDA is supposed to provide over the naive approach (§20) |

Comparing against a **hand-computable, exact** expected value — not just "does the image look plausible" — is what actually catches the sign errors and off-by-one mistakes in §66's list before they hide inside a complex, hard-to-debug scene.

---

# 68. The Live Ray Tracing Simulation (What You'll See in the HTML Version)

The HTML companion also embeds a **real, running software ray tracer** — vanilla JavaScript, no WebGL, rendering directly into a `<canvas>` pixel by pixel. It implements the sphere and plane intersection math from §31–§33, the Phong/Blinn-Phong shading from §39–§42, shadow rays from §43, and one level of recursive reflection from §45–§46, at a resolution and sample count tuned to finish in well under a second in a browser — click "Render" and watch the same mathematics from §29–§49 produce an actual image, line by line, in real time. Refraction and the full Fresnel blend (§47–§48) are described in this document's math but intentionally left out of the *live* demo to keep it fast enough to feel interactive; the offline Java renderer in §51–§59 implements the complete model, glass sphere included.

---

# 69. Final Architecture

```text
                          Camera (§16-17)
                                |
                    getRay(s, t) per pixel (§52)
                                |
                                v
                    RayTracer.traceRay(ray, depth) (§49)
                                |
                    Scene.hitNearest(ray) (§51)
                                |
                    BVH (§61-62) -> Hittable primitives
                    Sphere (§31-32) / Plane (§33) / Triangle (§34-36)
                                |
                    HitRecord: point, normal, material
                                |
              +-----------------+-----------------+
              v                                   v
    Local shading (§37-44)              Recursive secondary rays
    ambient + diffuse + specular         reflection (§45-46)
    + shadow rays                        refraction (§47-48, §50)
              |                                   |
              +-----------------+-----------------+
                                v
                    Blended final pixel color
                                |
                    Supersampling average (§57)
                                |
                    PPM file (§58) or <canvas> pixel (§68)
```

Separately, the 2D ray caster (§18-§28) is architecturally simpler and independent of all of the above — it shares only the ray equation (§13-§14) and the "one ray, find the nearest hit, use the distance" shape with its 3D sibling.

---

# 70. Suggested V2 Enhancements

| Enhancement | What it adds | Where it plugs in |
|---|---|---|
| Texture mapping | Wood grain, brick patterns, images wrapped onto surfaces instead of flat colors | Requires per-primitive UV coordinates (barycentric-interpolated for triangles, §35) mapped into a 2D image lookup, replacing `material.color` with a texture sample |
| Perlin/procedural noise textures | Marble, wood, clouds, without a texture image file | A noise function evaluated at the hit point's world-space coordinates, feeding into the material's color or bump-mapped normal |
| Bump/normal mapping | Surface detail (bumps, wrinkles) without added geometry | Perturbing the shading normal (§36) using a lookup texture, without changing the true geometric surface |
| Depth of field | Realistic camera focus blur | Jittering the camera's ray *origin* across a small lens aperture (in addition to §57's per-pixel jitter), converging rays through a focal point |
| Path tracing / global illumination | Indirect lighting, color bleeding, soft area-light shadows without explicit sampling code (§64) | The substantially larger extension outlined conceptually in §65 |
| GPU acceleration (compute shaders) | Orders-of-magnitude faster rendering by tracing millions of rays in parallel | Porting §52's per-pixel loop to run on the GPU, one thread per pixel (or per ray) |
| Constructive Solid Geometry (CSG) | Objects built by boolean union/intersection/difference of primitives | Combining multiple `t` intervals per object (entry/exit pairs) with boolean interval logic before picking the nearest valid surface |
| Volumetric rendering (fog, smoke) | Light scattering *within* a volume, not just at surfaces | Requires ray-marching through a density field rather than a single surface intersection per ray |

---

# 71. Progressive Interview/Practice Question Set

**Level 1 — Vector foundations**
1. Explain, geometrically, what the dot product and cross product each compute, and why one uses cosine and the other sine.
2. Why must a ray's direction be normalized for the ray-sphere intersection formula in §31 to work correctly as written?

**Level 2 — Ray casting**
3. Walk through the DDA algorithm step by step for a ray pointed diagonally through a grid, tracking `sideDistX`/`sideDistY` by hand for the first three steps.
4. Why does ray casting look distorted without the perpendicular-distance correction, and why does DDA compute that correction "for free"?

**Level 3 — Intersection math**
5. Derive the ray-sphere intersection quadratic from the sphere's implicit equation and the ray equation, from scratch.
6. What does a negative discriminant mean geometrically, and what does exactly one root (discriminant = 0) mean?

**Level 4 — Shading**
7. Explain why Lambertian diffuse shading depends only on the light direction and normal, never the viewer's position, while specular shading depends on both.
8. Why is a shadow ray's origin nudged slightly off the surface before it's traced?

**Level 5 — Reflection, refraction, recursion**
9. Derive the mirror reflection formula from first principles (decomposing a vector relative to a normal).
10. What does total internal reflection mean physically, and how does a ray tracer's code detect it?
11. Why does a recursive ray tracer need an explicit depth limit, and what happens to image quality as that limit increases?

**Level 6 — Performance**
12. Explain what a bounding volume hierarchy does to reduce ray-scene intersection cost, and why testing a bounding box first is cheap.
13. Why is ray tracing considered "embarrassingly parallel," and what's the one part of a naive parallel implementation that still needs synchronization?

**Final challenge:** Design (in words, with the relevant formulas named) a renderer feature that shows a colored, slightly blurred reflection of a diffuse red wall on a glossy (not perfectly mirror-smooth) metal sphere — identify which of this guide's sections' formulas you'd combine, and where you'd introduce randomness to produce the "glossy" (rather than perfectly sharp) blur.

---

# 72. Final Takeaway

Both algorithms in this guide reduce, ultimately, to the same equation repeated at different scales: $P(t) = O + t\vec{D}$ (§13–§14). Ray casting asks that equation "which grid cell do you cross next" (§20–§23); ray tracing asks it "which exact 3D surface do you intersect, and at what geometric angle" (§30–§36), then layers a physically-motivated lighting model (§37–§44) and a recursive bouncing scheme (§45–§50) on top of the *same* underlying question. Every piece of "magic" in a renderer — a wall that shrinks correctly with distance, a highlight that appears in exactly the right place, a glass sphere that both reflects and refracts — is one specific, derivable formula from this guide, not an unexplainable rule. Whether you build the version in this document in Java, port it to another language, or extend the live JavaScript simulations in the companion HTML file, the underlying mathematics — vectors, dot products, quadratics, and the law of reflection — is exactly the same mathematics inside every production renderer, from a 1992 shareware game to a modern GPU path tracer.

