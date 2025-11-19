# Problems with surfing physics in the Source Engine and how Momentum Mod solved them

## Summary

This document provides a comprehensive technical analysis of surfing mechanics in the Source Engine, examining both the vanilla Source SDK 2013 implementation and the bug-fixed enhancements found in Momentum Mod.

**Note**: This information contained within this draft may not be fully accurate, due to either my misunderstanding or insufficient proof reading. It should not be considered fact and all claims should be cross referenced against sources.

**Version Information**: Draft 3.0 | 19/11/2025

**To Do**: 
- Add more code snippets 
- Improve readability 
- Add visualization 
- Audit and ensure accuracy 
- Improve formatting 
- Add links and citations

**Source Code References**
- [Source Engine 2018 Leak HL2/TF2 branch - gamemovement.cpp](https://github.com/followingthefasciaplane/source-engine-diff-check/blob/main/collection/gamemovement_2018_hl2.cpp)
- [Momentum Mod - mom_gamemovement.cpp](https://github.com/momentum-mod/game/blob/develop/mp/src/game/shared/momentum/mom_gamemovement.cpp)

---

## Table of Contents

1. [Introduction and Core Concepts](#introduction-and-core-concepts)
2. [Surface Geometry and the Steepness Rule](#surface-geometry-and-the-steepness-rule)
3. [Fundamental Physics: Gravity and Velocity Clipping](#fundamental-physics-gravity-and-velocity-clipping)
4. [Terminology Reference](#terminology-reference)
5. [The Movement Loop: TryPlayerMove](#the-movement-loop-tryplayermove)
6. [Velocity Projection: ClipVelocity](#velocity-projection-clipvelocity)
7. [Air Control: AirMove and AirAccelerate](#air-control-airmove-and-airaccelerate)
8. [Gravity Integration: StartGravity and FinishGravity](#gravity-integration-startgravity-and-finishgravity)
9. [Ground Categorization and Edge Detection](#ground-categorization-and-edge-detection)
10. [Step Movement and Slope Handling](#step-movement-and-slope-handling)
11. [Complete Frame Execution Flow](#complete-frame-execution-flow)
12. [Momentum Mod Improvements Summary](#momentum-mod-improvements-summary)

---

## Introduction and Core Concepts

### What is Surfing?

Surfing is a player movement technique that exploits the Source Engine's collision resolution system to glide along sloped surfaces and gain speed. When a player contacts a surface that is too steep to stand on but not vertical, the engine resolves the collision by projecting the player's velocity along the surface plane. Combined with gravity's constant downward acceleration, this projection converts potential energy (falling) into kinetic energy (horizontal speed).

The technique requires surfaces that meet specific geometric criteria and relies on the interplay between several core engine systems: collision detection, velocity adjustment, air acceleration, and gravity application. Understanding surfing requires examining both the mathematical principles underlying these systems and their specific implementation in the Source Engine codebase.

### Why Two Implementations?

The vanilla Source implementation (used in Counter-Strike: Source, Team Fortress 2, and Half-Life 2) contains several physics bugs that cause random, frustrating behavior during surfing. Players might suddenly stop mid-surf when crossing ramp seams, get stuck inside geometry due to floating-point errors, or lose speed unpredictably when boarding ramps. These issues stem from hardcoded collision resolution limits, inadequate handling of numerical precision errors, and velocity projection methods that don't account for high-speed movement.

Momentum Mod addresses these issues through targeted fixes while preserving the core physics that define Source Engine movement. The project introduces configurable parameters for collision resolution, retrace logic to handle floating-point errors, specialized handling for parallel plane collisions, and velocity preservation techniques for slope interactions. These improvements eliminate randomness while maintaining the skill-based nature of surfing.

---

## Surface Geometry and the Steepness Rule

### The Mathematical Threshold

For the engine to treat a surface as surfable rather than walkable, it must satisfy a specific geometric constraint defined by the surface normal vector. A surface normal is a unit vector perpendicular to the surface, pointing away from the solid material. The engine uses the Z component (vertical component) of this normal to categorize surfaces.

**The Rule**: A surface is surfable if and only if the Z component of its normal vector is less than 0.7.

This threshold derives from the relationship between the normal vector and the angle of the surface relative to the horizontal plane. A perfectly flat floor has a normal of (0, 0, 1), meaning the Z component equals 1. A vertical wall has a normal where Z equals 0. The threshold of 0.7 represents the cosine of the critical angle: arccos(0.7) ≈ 45.57 degrees.

Therefore, any surface steeper than approximately 45.5 degrees relative to horizontal qualifies as a surf ramp. The engine treats the player as airborne when touching such surfaces, which has three critical implications. First, friction is not applied to the player's velocity. Second, the player receives full air control through the air acceleration system. Third, the collision is resolved using `ClipVelocity` rather than ground movement logic.

### Engine State Determination

The function `CategorizePosition` runs every frame to determine whether the player should be in a grounded or airborne state. This determination affects which movement code path executes and fundamentally alters how velocity changes are calculated. The categorization process involves tracing downward from the player's position to detect nearby surfaces and examining the normal vector of any surface found.

For surfing to work, the engine must maintain the airborne state despite continuous contact with the ramp surface. The steep angle ensures this because the normal check (normal.z < 0.7) fails to qualify the surface as ground. However, the vanilla engine has an issue where high-speed upward movement on slopes can cause premature ground-snapping, instantly applying friction and killing momentum.

**Momentum Mod Enhancement**: The `CategorizePosition` function in Momentum Mod includes predictive logic. It traces ahead one tick to determine where the player will be on the next frame. If this prediction shows the player sliding up the ramp at high speed or falling off an edge, the function maintains the airborne state even if the player is technically close to a surface. This prevents friction application during valid surf movement and enables techniques like edge-bugging, where players can catch the edge of a surface without being snapped to the ground.

---

## Fundamental Physics: Gravity and Velocity Clipping

### The Core Interaction

Surfing emerges from the continuous interaction between two opposing forces: gravity pulling the player downward and collision resolution pushing the player away from surfaces. Understanding this interaction requires examining both the vector mathematics involved and the frame-by-frame application of these forces.

Every frame, gravity adds negative Z velocity to the player. This downward acceleration is constant, providing a reliable force that would normally cause the player to fall. However, when the player's velocity vector intersects with a ramp surface, the collision detection system triggers. The engine cannot allow penetration of solid geometry, so it must adjust the velocity to redirect motion along the surface rather than into it.

This adjustment happens through the `ClipVelocity` function, which performs a vector projection operation. The function calculates how much velocity is directed into the surface (the dot product of velocity and surface normal) and subtracts that component from the velocity vector. What remains is only the velocity parallel to the surface.

Here's the critical insight: because gravity is purely vertical (0, 0, -G) and the ramp normal has both horizontal and vertical components, removing the perpendicular component doesn't eliminate all the gravitational velocity. Instead, it redirects the energy. The vertical velocity gets partially converted into horizontal velocity aligned with the ramp's slope.

### Energy Conversion Mechanics

Consider a ramp angled at 45 degrees facing in the positive X direction. Its normal vector is approximately (0.707, 0, 0.707). When gravity has added velocity of (0, 0, -10) to the player, the ClipVelocity operation removes the component perpendicular to the ramp. The dot product of (0, 0, -10) with (0.707, 0, 0.707) equals approximately -7.07. This backoff value gets multiplied by the normal and subtracted from the incoming velocity.

The calculation proceeds as follows. The backoff vector equals -7.07 times (0.707, 0, 0.707), which equals approximately (-5, 0, -5). Subtracting this from the incoming velocity (0, 0, -10) yields (0 - (-5), 0 - 0, -10 - (-5)) equals (5, 0, -5). The player now has positive X velocity that wasn't present before gravity's application. This is the fundamental mechanism of surfing: gravitational energy continuously converts into horizontal motion through repeated ClipVelocity operations.

The beauty of this system is its sustainability. As long as the player remains on the ramp, gravity continues adding downward velocity every frame, and ClipVelocity continues converting it. This creates a positive feedback loop where speed accumulates over time. Air acceleration allows players to control their trajectory along the ramp, steering left or right while maintaining or increasing speed.

---

## Terminology Reference

Understanding the technical implementation requires familiarity with specific terms used throughout the Source Engine codebase. These terms represent mathematical concepts, physical quantities, and engine-specific variables that control movement behavior.

**Velocity** refers to the player's current speed and direction in three-dimensional space, represented as a vector with X, Y, and Z components. In surfing contexts, "units" often refers specifically to the magnitude of the horizontal components (X and Y), while velocity encompasses the complete three-dimensional vector including vertical motion.

**Wishdir** is the normalized direction vector representing where the player intends to move based on their input and view direction. The engine calculates this by combining the player's forward and right vectors (derived from view angles) with their movement inputs, then normalizing the result to unit length.

**Wishspeed** is the scalar magnitude representing how fast the player intends to move in the wishdir direction. This value combines forward and sideways input magnitudes and gets clamped to maximum speed limits.

**Wishvel** is the complete desired velocity vector calculated by multiplying wishdir by wishspeed. It represents both the direction and magnitude of intended movement.

**Air Max Wishspeed** is a hardcoded cap on wishspeed during airborne movement, typically set to 30 units. This limit is crucial for preventing infinite acceleration when holding forward and forms the foundation of strafe acceleration mechanics. Without this cap, players could gain unlimited speed simply by holding forward, eliminating the skill requirement of proper air strafing.

**Currentspeed** (or veer) is the component of the player's existing velocity that lies in the wishdir direction. This scalar value is calculated via dot product and determines how much additional acceleration can be applied. When currentspeed approaches or exceeds the wishspeed cap, acceleration diminishes or stops entirely.

**Addspeed** represents the difference between the desired speed (wishspeed) and the current speed in that direction (currentspeed). This is the maximum amount of velocity that can be added this frame before hitting the speed cap.

**Accelspeed** is the actual acceleration applied this frame, calculated by multiplying the acceleration coefficient (sv_airaccelerate), wishspeed, frametime, and surface friction. This value gets capped by addspeed to prevent overshooting the target velocity.

**Surface Normal** is a unit vector perpendicular to a surface, pointing away from solid material. For collision resolution, this vector defines the plane of the surface and determines how velocity should be adjusted. The Z component of the normal determines whether a surface is classified as floor (Z > 0.7), surf ramp (0 < Z < 0.7), or wall (Z ≈ 0).

**Backoff** in the context of ClipVelocity is the magnitude of velocity directed into a surface that must be removed to prevent penetration. It's calculated as the dot product of incoming velocity and surface normal, scaled by the overbounce factor.

**Overbounce** is a multiplier that determines how much of the perpendicular velocity component should be removed. For normal surfaces including surf ramps, this value is 1.0, meaning the perpendicular component is completely removed. Values greater than 1.0 create bouncy surfaces, while values less than 1.0 create surfaces that absorb momentum.

**Origin** is the player's current position in world space, specified by X, Y, and Z coordinates. Movement calculations modify this position each frame based on velocity and collision results.

**Frametime** (gpGlobals->frametime) is the time interval between server ticks, typically 0.015 seconds for 66-tick servers or 0.01 seconds for 100-tick servers. All physics calculations scale by frametime to ensure consistent behavior regardless of tick rate. Momentum Mod enforces fixed timesteps to prevent physics variance caused by server FPS fluctuations.

---

## The Movement Loop: TryPlayerMove

The `TryPlayerMove` function represents the core collision resolution system in the Source Engine. This function executes every frame and handles the complex task of moving the player through the world while respecting solid geometry. Understanding this function is essential for grasping both why surfing works and why certain bugs occur in the vanilla engine.

### Algorithm Structure

The function operates through an iterative loop that attempts to move the player along their velocity vector while handling any collisions encountered. The loop structure allows for multiple collision resolutions per frame, which is necessary when the player's trajectory intersects multiple surfaces or complex geometry.

At initialization, the function sets up critical variables including the bump counter, plane storage array, velocity copies, and time remaining for this frame. The bump counter limits how many collision resolution iterations can occur, preventing infinite loops in edge cases. Plane storage maintains a record of all surfaces contacted this frame, which becomes important when handling multiple simultaneous collisions.

The core loop begins by checking if the player still has velocity. If velocity has been reduced to zero by previous collision resolutions, the loop exits early. Otherwise, the function calculates an endpoint based on the current position plus velocity scaled by the remaining time. A trace operation then checks for collision along this path.

### Trace Operation and Results

The trace operation returns comprehensive information about any collision encountered. The trace fraction indicates what percentage of the desired movement completed before hitting something. A fraction of 1.0 means no collision occurred and the full movement succeeded. A fraction of 0.5 means the player traveled halfway to the target before hitting a surface. A fraction of 0.0 indicates the player started inside solid geometry, which represents a critical error state.

When the trace detects a collision, it provides the exact impact point, the surface normal of the contacted geometry, and flags indicating whether the trace started in solid material. The vanilla engine has limited handling for traces that start in solid, often simply zeroing velocity and stopping the player. This is the root cause of the "ramp bug" where players randomly stop on complex surf ramps.

### Vanilla Implementation Limitations

The vanilla Source SDK hardcodes the bump limit to 4, meaning the collision resolution loop can execute at most four times per frame. This seemed sufficient for typical gameplay scenarios involving simple geometry like floors, walls, and stairs. However, complex curved surf ramps can easily generate more than four collision points per frame, especially at high speeds.

When the bump limit is reached before all movement time is consumed, the function forcibly exits, leaving the player at their current position with remaining velocity intact. On the next frame, movement continues from this stopped position, but momentum may have been lost or redirected unpredictably. This manifests as stuttering or complete stops mid-surf, seemingly without cause.

Another vanilla issue involves the handling of traces that start inside solid geometry. When floating-point precision errors accumulate (which happens frequently in complex collision scenarios), a trace might begin slightly inside a surface. The vanilla code detects this condition (pm.allsolid equals true) and responds by zeroing all velocity and returning immediately. The player stops dead, often appearing to be stuck in mid-air.

### Momentum Mod: Expanded Resolution and Ramp Fix

Momentum Mod addresses these limitations through several targeted improvements. First, the bump limit becomes configurable via the ConVar `sv_ramp_bumpcount`, defaulting to 8 rather than 4. This simple change alone eliminates most random stops on complex geometry by allowing sufficient iterations to resolve all collisions.

More significantly, Momentum implements a "ramp fix" system that handles the floating-point error case gracefully. Rather than immediately stopping when a trace starts in solid, the code sets a flag indicating the player is stuck on a ramp. On the next loop iteration, special logic activates.

The ramp fix attempts to find a valid plane to slide along by searching a 3x3x3 grid of directions around the player. For each direction, it performs a short trace and examines whether that direction provides a valid exit from the stuck state. Once a valid plane is found, the code "retraces" the player's origin backward along that plane by a small amount (controlled by `sv_ramp_initial_retrace_length`). This effectively pulls the player out of the geometry, allowing normal collision resolution to resume.

Here's the conceptual flow in Momentum Mod:

```cpp
// Inside the main bump loop
if (stuck_on_ramp && sv_ramp_fix.GetBool())
{
    // If we have valid plane information from a previous iteration
    if (has_valid_plane)
    {
        // Clip velocity against the plane we were stuck on
        ClipVelocity(mv->m_vecVelocity, valid_plane, mv->m_vecVelocity, 1);
    }
    else
    {
        // Search for a valid plane by tracing in multiple directions
        // This involves checking 27 different directional vectors
        // to find one that leads out of the stuck geometry
        for (int i = -1; i <= 1; i++)
            for (int j = -1; j <= 1; j++)
                for (int k = -1; k <= 1; k++)
                {
                    // Construct direction, trace, check validity
                    // Store in valid_plane if found
                }
    }
    
    // Retrace the origin backward to free the player
    // This moves the player slightly away from the stuck position
    VectorMA(fixed_origin, sv_ramp_initial_retrace_length.GetFloat(), 
             valid_plane, fixed_origin);
}
```

### The Vertical Ramp Bug Fix

Another critical Momentum Mod improvement addresses what's known as the "vertical ramp bug." In vanilla Source, when a player collides with a crease (where two planes meet), the engine attempts to calculate the direction along that crease using the cross product of the two plane normals. This calculation assumes the planes intersect at an angle, creating a line along which the player should slide.

However, in surf maps, ramps are often constructed from multiple brush entities that meet perfectly flush. At these seams, the player can contact both brushes simultaneously, registering two planes with identical or nearly identical normals. When the normals are parallel or nearly parallel, the cross product operation fails or produces an unstable result, causing velocity to zero out or redirect unpredictably.

Momentum Mod detects this condition by comparing the two plane normals with a tolerance check. If the planes are effectively identical (CloseEnough returns true), the code skips the cross product calculation entirely. Instead, it pushes the player slightly away from the duplicate plane, preserving their horizontal velocity while preventing the stuck state:

```cpp
// When we have exactly two collision planes
if (numplanes == 2 && CloseEnough(planes[0], planes[1], FLT_EPSILON))
{
    // The planes are identical or nearly parallel
    // Push the player away from the surface slightly
    VectorMA(original_velocity, 20.0f, planes[0], new_velocity);
    
    // Preserve horizontal velocity but not vertical
    // This prevents gaining height advantages from the bug fix
    mv->m_vecVelocity.x = new_velocity.x;
    mv->m_vecVelocity.y = new_velocity.y;
    // Note: Z velocity is preserved from previous calculation
    
    break; // Exit the loop to prevent further processing
}
```

This elegant solution maintains smooth movement across ramp seams without allowing exploit behavior like gaining free height.

---

## Velocity Projection: ClipVelocity

The `ClipVelocity` function performs the mathematical operation that makes surfing possible. This function adjusts velocity vectors when collisions occur, ensuring players slide along surfaces rather than penetrating them. The operation it performs is a vector projection that removes the component of velocity directed into the surface while preserving components parallel to it.

### Mathematical Foundation

Vector projection is a fundamental operation in linear algebra. Given an incoming velocity vector **v** and a surface normal **n**, we want to find the component of **v** that lies perpendicular to **n** and remove it. The magnitude of this perpendicular component is given by the dot product **v** · **n**.

The dot product calculates how much of one vector points in the direction of another. For a normalized surface normal, this dot product directly gives the magnitude of velocity directed into (or away from) the surface. Multiplying this scalar by the normal vector itself produces the vector component we need to remove.

The adjusted velocity **v'** is calculated as **v'** = **v** - (**v** · **n**)**n**. This formula subtracts the perpendicular component from the original velocity, leaving only the parallel component. The overbounce factor scales this removal, typically set to 1.0 for normal surfaces.

### Vanilla Implementation

The vanilla Source SDK implementation is straightforward but has implications for high-speed movement:

```cpp
int CGameMovement::ClipVelocity(Vector& in, Vector& normal, Vector& out, float overbounce)
{
    float backoff;
    float change;
    int i;
    int blocked;
    
    // Determine surface type based on normal Z component
    float angle = normal[2];
    blocked = 0x00;  // Assume unblocked
    
    if (angle > 0)
        blocked |= 0x01;  // Floor surface (can stand on)
    if (!angle)
        blocked |= 0x02;  // Wall surface (vertical)
    
    // Calculate how much velocity goes into the surface
    // This is the dot product of incoming velocity and surface normal
    backoff = DotProduct(in, normal) * overbounce;
    
    // Remove that amount from each component of velocity
    for (i = 0; i < 3; i++)
    {
        change = normal[i] * backoff;
        out[i] = in[i] - change;
    }
    
    // Ensure we're not still moving into the surface
    // This catches floating-point precision errors
    float adjust = DotProduct(out, normal);
    if (adjust < 0.0f)
    {
        out -= (normal * adjust);
    }
    
    return blocked;
}
```

The final precision check is crucial. Due to floating-point arithmetic limitations, the calculated output velocity might still have a tiny component directed into the surface. The adjustment step ensures the output velocity is exactly parallel to or pointing away from the surface.

### The Slope Fix Problem

In the vanilla engine, ClipVelocity's strict adherence to vector projection creates problems when players try to board ramps from above or jump up slopes. Consider a player moving horizontally at high speed who jumps and lands on an upward slope. Their velocity vector has large horizontal components and a downward vertical component.

When this velocity is clipped against the slope's normal, the projection removes not just the perpendicular component but effectively reduces the horizontal magnitude. This happens because the slope's normal has horizontal components, and the dot product captures both the downward velocity and some of the horizontal velocity as being "into" the surface.

Mathematically, this is correct behavior for a collision. However, from a gameplay perspective, it feels wrong. The player expects to maintain their horizontal speed while sliding up the ramp, not to suddenly slow down. This speed loss makes it difficult to board ramps smoothly and punishes aggressive movement.

### Momentum Mod: Slope Fix

Momentum Mod's slope fix addresses this by adding special logic to ClipVelocity that preserves horizontal velocity under specific conditions. The fix only activates when the player has autobhop enabled and is holding jump, and only when landing on surfaces that qualify as floor (normal.z >= 0.7) with reasonable vertical velocity.

The key insight is that if the clipping operation would reduce the player's horizontal speed below what they had before the collision, and they're trying to jump, the fix restores the original horizontal velocity components while zeroing the vertical component:

```cpp
int CMomentumGameMovement::ClipVelocity(Vector& in, Vector& normal, Vector& out, float overbounce)
{
    // Perform standard clipping calculation
    const int blocked = BaseClass::ClipVelocity(in, normal, out, overbounce);
    
    // Check if RNG fix system is enabled
    if (sv_rngfix_enable.GetBool()) return blocked;
    
    // Slope fix: preserve horizontal velocity when jumping up inclines
    if (sv_slope_fix.GetBool() && m_pPlayer->HasAutoBhop() && (mv->m_nButtons & IN_JUMP))
    {
        // Check if this is a surface we could potentially stand on
        // and that we're not flying upward too fast
        bool canJump = normal[2] >= 0.7f && out.z <= NON_JUMP_VELOCITY;
        
        // Check if we're moving into the slope horizontally
        // and if clipping reduced our horizontal speed
        bool movingIntoSlope = (normal.x * in.x + normal.y * in.y < 0.0f);
        bool lostSpeed = (out.Length2DSqr() <= in.Length2DSqr());
        
        if (canJump && movingIntoSlope && lostSpeed)
        {
            // Restore original horizontal velocity
            out.x = in.x;
            out.y = in.y;
            // But keep vertical velocity at zero to prevent flying
            out.z = 0.0f;
        }
    }
    
    return blocked;
}
```

This modification allows players to smoothly board ramps while bunnyhoping without the jarring speed loss of vanilla. It doesn't grant free speed but rather prevents the loss of existing momentum when transitioning onto slopes.

---

## Air Control: AirMove and AirAccelerate

Air movement in Source games feels distinctly different from traditional first-person shooters. Players have significant control over their trajectory while airborne, able to accelerate, decelerate, and change direction without touching the ground. This control system forms the foundation of advanced movement techniques including surfing, bunnyhopping, and air strafing.

### AirMove: Input Processing

The `AirMove` function serves as the orchestrator for airborne movement. It runs every frame when the player is not grounded and handles the conversion of player input into desired movement direction and speed. The function's primary responsibilities include calculating the wish direction from view angles and input, determining wish speed, calling the acceleration function, and executing movement with collision handling.

The function begins by extracting forward, right, and up vectors from the player's view angles. These vectors define the player's local coordinate frame based on where they're looking. The forward vector points in the direction the player faces, the right vector points to their right side, and the up vector points directly upward in their local space.

Movement input values (forward and sideways) are then applied to these vectors. A critical step occurs next: the Z components of the forward and right vectors are forcibly set to zero and the vectors are renormalized. This constraint is fundamental to Source air control because it ensures that looking up or down doesn't affect horizontal movement speed.

Consider the implications of this design choice. Without zeroing the Z component, looking straight down while holding forward would produce a wish vector pointing downward, potentially exceeding the air speed cap when combined with gravity. By constraining movement input to the horizontal plane, the engine separates vertical control (handled by gravity and jump mechanics) from horizontal control (handled by player input).

The wish velocity is calculated by combining the constrained forward and right vectors with input magnitudes:

```cpp
void CGameMovement::AirMove(void)
{
    Vector wishvel;
    float fmove, smove;
    Vector wishdir;
    float wishspeed;
    Vector forward, right, up;
    
    // Convert view angles to direction vectors
    AngleVectors(mv->m_vecViewAngles, &forward, &right, &up);
    
    // Get movement inputs (typically -450 to +450 for walking speed)
    fmove = mv->m_flForwardMove;
    smove = mv->m_flSideMove;
    
    // Constrain movement to horizontal plane
    // This is critical for proper air control
    forward[2] = 0;
    right[2] = 0;
    VectorNormalize(forward);
    VectorNormalize(right);
    
    // Calculate desired velocity in world space
    for (int i = 0; i < 2; i++)
        wishvel[i] = forward[i] * fmove + right[i] * smove;
    wishvel[2] = 0;  // Explicitly zero vertical component
    
    // Extract direction and magnitude
    VectorCopy(wishvel, wishdir);
    wishspeed = VectorNormalize(wishdir);
    
    // Clamp to maximum allowed speed
    if (wishspeed != 0 && wishspeed > mv->m_flMaxSpeed)
    {
        VectorScale(wishvel, mv->m_flMaxSpeed / wishspeed, wishvel);
        wishspeed = mv->m_flMaxSpeed;
    }
    
    // Apply acceleration toward the wish direction
    AirAccelerate(wishdir, wishspeed, sv_airaccelerate.GetFloat());
    
    // Handle base velocity (from moving platforms, etc.)
    VectorAdd(mv->m_vecVelocity, player->GetBaseVelocity(), mv->m_vecVelocity);
    
    // Execute movement with collision handling
    TryPlayerMove();
    
    // Remove base velocity after movement
    VectorSubtract(mv->m_vecVelocity, player->GetBaseVelocity(), mv->m_vecVelocity);
}
```

### AirAccelerate: The Core of Strafe Acceleration

The `AirAccelerate` function implements the algorithm that enables strafe jumping and surf control. Understanding this function is essential for grasping why looking sideways while pressing forward produces acceleration, a behavior that seems counterintuitive but forms the basis of advanced Source movement.

The function's strategy involves calculating how much the player is currently moving in the desired direction, determining how much additional speed can be added without exceeding limits, computing an acceleration amount based on game settings and frametime, and applying that acceleration in the wish direction.

The critical insight lies in the concept of currentspeed, which measures velocity already present in the wish direction via dot product. When the player looks forward while moving forward at high speed, currentspeed is high and approaches or exceeds the air speed cap of 30 units. This means addspeed becomes zero or negative, preventing further acceleration.

However, when the player looks 90 degrees sideways while maintaining forward input, the wish direction becomes perpendicular to their current velocity. The dot product of perpendicular vectors is zero, so currentspeed equals zero regardless of how fast they're moving forward. This allows the full 30 units of addspeed to be applied in the sideways direction.

Through vector addition, adding sideways velocity to forward velocity produces a resultant velocity with greater magnitude. This is the Pythagorean theorem in action: if you have 1000 units forward velocity and add 30 units sideways velocity, your new speed is sqrt(1000² + 30²) ≈ 1000.45 units. The speed increase is small per frame but accumulates rapidly.

The implementation carefully clamps acceleration to prevent exceeding the calculated addspeed limit:

```cpp
void CGameMovement::AirAccelerate(Vector& wishdir, float wishspeed, float accel)
{
    float wishspd = wishspeed;
    
    // Early exits for special cases
    if (player->pl.deadflag || player->m_flWaterJumpTime)
        return;
    
    // Cap the desired speed to the air speed limit (typically 30)
    // This is THE fundamental limit that makes strafe acceleration work
    if (wishspd > GetAirSpeedCap())
        wishspd = GetAirSpeedCap();
    
    // Calculate how much we're already moving in the desired direction
    // This is the key to understanding why strafing sideways works
    float currentspeed = mv->m_vecVelocity.Dot(wishdir);
    
    // Determine how much we can add before hitting the cap
    float addspeed = wishspd - currentspeed;
    
    // If we're already at or above the cap in this direction, add nothing
    if (addspeed <= 0)
        return;
    
    // Calculate actual acceleration amount for this frame
    // This scales with sv_airaccelerate, wishspeed, frametime, and friction
    float accelspeed = accel * wishspeed * gpGlobals->frametime * player->m_surfaceFriction;
    
    // Don't exceed the calculated addspeed limit
    if (accelspeed > addspeed)
        accelspeed = addspeed;
    
    // Apply acceleration in the wish direction
    for (int i = 0; i < 3; i++)
    {
        mv->m_vecVelocity[i] += accelspeed * wishdir[i];
        mv->m_outWishVel[i] += accelspeed * wishdir[i];
    }
}
```

### Surf Application

During surfing, AirAccelerate runs continuously while the player is on the ramp. The player uses this system to control their trajectory along the ramp surface, steering left or right to follow the ramp's contours. The same strafing technique that produces bunnyhopping works on ramps, allowing players to optimize their path and maximize speed gain.

The interaction between air acceleration and ClipVelocity creates the characteristic feel of surfing. Air acceleration allows building speed in specific directions, ClipVelocity converts vertical velocity from gravity into horizontal velocity along the ramp, and the combination produces smooth, controllable high-speed movement. Skilled players learn to balance their view angle, strafe direction, and timing to maintain optimal acceleration while following the ramp's path.

---

## Gravity Integration: StartGravity and FinishGravity

Gravity in the Source Engine doesn't apply as a single instantaneous force each frame. Instead, it uses a numerical integration technique called the leapfrog method (also known as symplectic Euler integration) that splits gravity application into two half-steps. This approach provides better stability and accuracy for physics simulations compared to applying the full force at once.

### Why Split Integration?

Numerical integration methods for solving differential equations involve trade-offs between accuracy, stability, and computational cost. Simple Euler integration (applying forces all at once) is fast but accumulates error over time and can become unstable at large timesteps. Runge-Kutta methods provide better accuracy but require multiple force evaluations per step, increasing computational cost.

The leapfrog method occupies a middle ground. By staggering velocity and position updates, it maintains time-reversibility and conserves energy better than simple Euler integration. This matters for game physics because players expect consistent, predictable behavior. Velocity should increase linearly under constant acceleration, not drift unpredictably due to numerical errors.

The implementation splits each frame's gravity into two applications: half at the start of movement processing and half at the end. Between these applications, all collision detection, velocity clipping, and air acceleration occurs. This interleaving ensures that gravity affects the player's trajectory throughout the movement calculation rather than being applied in a single lump.

### StartGravity Implementation

The `StartGravity` function runs early in the movement processing pipeline, before collision detection and input handling. It applies half of the frame's gravitational acceleration and handles some base velocity bookkeeping:

```cpp
void CGameMovement::StartGravity(void)
{
    float ent_gravity;
    
    // Check if the player has custom gravity
    if (player->GetGravity())
        ent_gravity = player->GetGravity();
    else
        ent_gravity = 1.0;
    
    // Apply half of gravity for this frame
    // The 0.5 factor is the key to the leapfrog method
    mv->m_vecVelocity[2] -= (ent_gravity * GetCurrentGravity() * 0.5 * gpGlobals->frametime);
    
    // Add vertical component of base velocity (e.g., from moving platforms)
    mv->m_vecVelocity[2] += player->GetBaseVelocity()[2] * gpGlobals->frametime;
    
    // Zero out vertical base velocity since we've applied it
    Vector temp = player->GetBaseVelocity();
    temp[2] = 0;
    player->SetBaseVelocity(temp);
    
    // Ensure velocity stays within bounds
    CheckVelocity();
}
```

The gravity value retrieved by `GetCurrentGravity()` typically equals 800 units per second squared by default in Source games. The entity gravity multiplier allows individual players or entities to have modified gravity values for special effects or gameplay mechanics.

### FinishGravity Implementation

The `FinishGravity` function runs after all movement processing completes, applying the remaining half of gravity's effect:

```cpp
void CGameMovement::FinishGravity(void)
{
    float ent_gravity;
    
    // Don't apply if water jumping
    if (player->m_flWaterJumpTime)
        return;
    
    // Check for custom gravity
    if (player->GetGravity())
        ent_gravity = player->GetGravity();
    else
        ent_gravity = 1.0;
    
    // Apply the remaining half of gravity
    mv->m_vecVelocity[2] -= (ent_gravity * GetCurrentGravity() * gpGlobals->frametime * 0.5);
    
    // Ensure velocity stays within bounds
    CheckVelocity();
}
```

The water jump check prevents gravity from interfering with the special water exit mechanic. When a player is performing a water jump, they follow a predetermined arc that shouldn't be modified by normal gravity application.

### Implications for Surfing

The split gravity application is crucial for surfing mechanics because it ensures gravity consistently adds downward velocity throughout the frame. This downward velocity serves as the energy source that ClipVelocity converts into forward motion along ramps.

Consider a complete frame during surfing. At frame start, StartGravity adds downward velocity. The player's velocity vector now points slightly more downward than before. During movement processing, this velocity intersects with the ramp surface. ClipVelocity removes the component perpendicular to the ramp, converting some downward velocity into forward velocity along the ramp. At frame end, FinishGravity adds more downward velocity. This new downward velocity is ready to be converted on the next frame.

This continuous cycle creates sustainable acceleration. Unlike air strafing which is limited by the 30 unit wishspeed cap, gravity-derived acceleration has no such limit. As long as the ramp continues downward and the player maintains proper positioning, speed can increase indefinitely (until hitting engine velocity limits or running out of ramp).

### Momentum Mod: Gravity Disable Logic

Momentum Mod adds support for map-based gravity control through special trigger entities. Surf maps sometimes include sections where normal gravity should be disabled, such as anti-gravity zones or special movement challenges. The implementation checks for these triggers before applying gravity:

```cpp
void CMomentumGameMovement::StartGravity(void)
{
    // Check if inside a slide trigger that disables gravity
    if (m_pPlayer->m_CurrentSlideTrigger && 
        m_pPlayer->m_CurrentSlideTrigger->m_bDisableGravity)
        return;
    
    // Otherwise apply gravity normally
    BaseClass::StartGravity();
}

void CMomentumGameMovement::FinishGravity(void)
{
    // Same check for the second half
    if (m_pPlayer->m_CurrentSlideTrigger && 
        m_pPlayer->m_CurrentSlideTrigger->m_bDisableGravity)
        return;
    
    BaseClass::FinishGravity();
}
```

This enhancement allows map designers to create varied gameplay experiences while maintaining the standard surf physics in normal sections.

---

## Ground Categorization and Edge Detection

The `CategorizePosition` function determines whether the player should be treated as grounded or airborne. This determination affects which movement code executes and fundamentally changes how velocity modifications are calculated. The categorization process runs every frame and must handle edge cases like being close to but not quite touching ground, moving up steep slopes at high speed, and transitioning between ground and air states.

### Vanilla Ground Detection

The vanilla implementation traces downward from the player's position by a small distance (typically 2 units) to detect nearby ground. If the trace hits a surface with a normal Z component greater than 0.7, and the player's downward velocity is small, the engine sets the player's ground entity. Once grounded, friction applies, and the movement code switches from air movement to ground movement.

This simple approach works well for normal gameplay but creates problems for advanced movement techniques. When bunnyhopping, players time their jumps to leave the ground on the exact frame they touch down, avoiding friction. If the ground detection is too aggressive, players get snapped to the ground prematurely, applying unwanted friction and killing their momentum.

### The Slope/Edge Bug

A specific case causes particular frustration: moving up a slope at high speed while bunnyhopping. In vanilla Source, the downward ground detection trace can succeed even when the player is technically moving upward, if the slope beneath them is close enough. The engine snaps the player to the ground, applies friction, and drastically reduces their horizontal velocity.

This behavior is technically correct for walking up slopes but completely breaks bhop chains and makes boarding surf ramps inconsistent. Players develop elaborate techniques to avoid triggering this snap, such as aiming their view at specific angles or timing jumps precisely, but the fundamental issue remains.

### Momentum Mod: Predictive Categorization

Momentum Mod solves these issues through predictive ground detection. Rather than only checking the current position, the function simulates where the player will be on the next frame and makes categorization decisions based on that prediction. This forward-looking approach prevents premature ground-snapping while maintaining proper ground detection when players actually intend to land.

The enhanced logic includes several conditions and checks. First, it determines if the player is currently holding jump with autobhop enabled, indicating bhop intent. Second, it traces ahead one full frame based on current velocity to predict the next position. Third, it examines whether this predicted position would be in the air or would contact ground. Fourth, it checks if an edge bug situation exists where the player would miss the ground entirely next frame.

Here's the conceptual implementation:

```cpp
void CMomentumGameMovement::CategorizePosition(void)
{
    // Get player's current position and downward velocity
    Vector point = mv->GetAbsOrigin();
    Vector traceStart = point;
    
    // Trace downward by 2 units to check for ground
    Vector traceEnd = point;
    traceEnd.z -= 2.0f;
    
    trace_t pm;
    TracePlayerBBox(traceStart, traceEnd, PlayerSolidMask(), 
                    COLLISION_GROUP_PLAYER_MOVEMENT, pm);
    
    // Standard vanilla checks
    bool bGrounded = pm.DidHit() && pm.plane.normal.z >= 0.7f;
    
    // Momentum Mod: Edge fix logic
    if (sv_edge_fix.GetBool() && 
        !(m_pPlayer->HasAutoBhop() && (mv->m_nButtons & IN_JUMP)))
    {
        // Predict where player will be next frame
        Vector predictedPos;
        VectorMA(point, gpGlobals->frametime, mv->m_vecVelocity, predictedPos);
        
        // Trace downward from predicted position
        trace_t pmFall;
        TracePlayerBBox(predictedPos, 
                       predictedPos + Vector(0, 0, -2.0f),
                       PlayerSolidMask(),
                       COLLISION_GROUP_PLAYER_MOVEMENT, 
                       pmFall);
        
        // If we would miss the ground next frame, don't ground us now
        // This enables edge bugs
        if (!pmFall.DidHit())
        {
            bGrounded = false;
        }
    }
    
    // Momentum Mod: Slope fix logic
    if (sv_rngfix_enable.GetBool() && bGrounded)
    {
        if (sv_slope_fix.GetBool())
        {
            // Compare current horizontal velocity to what it would be if grounded
            // Only actually ground the player if it doesn't cause significant speed loss
            Vector currentVelHorizontal = mv->m_vecVelocity;
            currentVelHorizontal.z = 0;
            float currentSpeed2D = currentVelHorizontal.Length();
            
            // Simulate clipping velocity against the ground plane
            Vector testVelocity;
            ClipVelocity(mv->m_vecVelocity, pm.plane.normal, testVelocity, 1.0f);
            testVelocity.z = 0;
            float testSpeed2D = testVelocity.Length();
            
            // Only ground if we wouldn't lose significant horizontal speed
            if (testSpeed2D < currentSpeed2D * 0.95f)
            {
                bGrounded = false;
            }
        }
    }
    
    // Apply the ground state
    if (bGrounded)
        SetGroundEntity(&pm);
    else
        SetGroundEntity(nullptr);
}
```

This predictive approach eliminates the premature ground-snapping that plagues vanilla movement while maintaining proper ground detection for normal gameplay. Players can maintain momentum through slope transitions and successfully execute edge bugs without fighting the physics system.

### Edge Bug Mechanics

Edge bugs represent an advanced technique where players catch the very edge of a surface, momentarily contacting it without being fully grounded. In vanilla Source, this requires pixel-perfect positioning and timing due to aggressive ground detection. With Momentum's edge fix, the technique becomes more consistent because the prediction system recognizes when the player would fall off the edge next frame and avoids grounding them.

The edge bug is valuable in both surf and bhop gameplay. In surfing, it allows players to catch ramp edges at high speeds without losing momentum to friction. In bunnyhopping, it provides an additional way to maintain speed through difficult geometry. The technique still requires skill to execute but no longer feels random or inconsistent.

---

## Step Movement and Slope Handling

The `StepMove` function handles movement up small obstacles like stairs and curbs. In typical gameplay, players should be able to walk up steps smoothly without manual jumping. The function achieves this by attempting two different movement paths (stepping up versus sliding along the ground) and choosing whichever advances the player further.

### Vanilla Step Logic

The vanilla implementation performs a normal ground movement attempt first. If this fails to move the player significantly (likely because they hit a step), the function tries an alternate path. It temporarily raises the player's origin by the maximum step height (typically 18 units), performs movement at this elevated position, then traces downward to place the player on top of the obstacle.

The function then compares the results of both attempts. If the stepped movement advanced the player further, that position is used. Otherwise, the ground movement result stands. This comparison usually uses horizontal distance traveled as the metric.

However, the step-down portion of this logic creates an unintended side effect. When the player is placed on top of the step and the downward trace completes, their velocity is adjusted based on the surface they landed on. If this surface is sloped downward, the velocity gets projected onto that slope. This projection can introduce negative Z velocity, effectively pushing the player down when they just moved up.

### The Slope Problem

This velocity modification becomes problematic when players move up steep slopes at high speeds. The step logic interprets the slope as a series of small steps, attempts to help the player "step up" them, then projects their velocity onto the slope surface. The downward projection adds negative Z velocity that wasn't present in the player's original movement intent.

This manifests as a sticky, slow feeling when approaching surf ramps from below or when bunnyhopping up slopes. The player's horizontal momentum is maintained, but the added downward velocity can cause premature ground detection on subsequent frames, leading to friction application and speed loss.

### Momentum Mod: Step Fix

Momentum Mod modifies `StepMove` to prevent this problematic velocity modification using an early-exit strategy. The key insight is that if the initial ground slide (the standard move) results in significant upward velocity (greater than `NON_JUMP_VELOCITY`), the player is likely surfing or moving fast up a ramp. In this case, attempting the complex "Step Up" logic is unnecessary and potentially harmful to momentum.

Instead of running both the ground move and the step move and comparing them, Momentum Mod checks the result of the initial ground move. If the player is moving up fast enough, it accepts that result immediately and aborts the step logic:

```cpp
void CMomentumGameMovement::StepMove(Vector &vecDestination, trace_t &trace)
{
    // If RNG fix system is disabled, use vanilla behavior
    if (!sv_rngfix_enable.GetBool())
    {
        BaseClass::StepMove(vecDestination, trace);
        return;
    }
    
    // Attempt normal ground movement (sliding along the floor/ramp)
    Vector vecEndPos = vecDestination;
    TryPlayerMove(&vecEndPos, &trace);
    
    // Check if the slide move resulted in significant upward velocity
    // NON_JUMP_VELOCITY is approximately 140 units/sec
    if (mv->m_vecVelocity.z > NON_JUMP_VELOCITY)
    {
        // We are likely flying up a ramp or surfing
        // The standard slide move is correct; do not attempt to "step up"
        // Stepping up here would likely clip our velocity against the ramp "stairs"
        // and kill our speed.
        
        // Calculate step height for statistics if needed
        float flStepDist = mv->GetAbsOrigin().z - vecPos.z;
        if (flStepDist > 0.0f)
        {
            mv->m_outStepHeight += flStepDist;
        }
        return; // EARLY EXIT: Skip the rest of StepMove
    }
    
    // ... If velocity was low, proceed with standard Step Up logic ...
}
```

This modification ensures that steep slopes are treated as smooth slides rather than jagged steps when moving at speed, maintaining the player's intended trajectory and preventing the sticky behavior that plagues vanilla. Players can approach ramps confidently, knowing the movement system will interpret their intent correctly.

---

## Complete Frame Execution Flow

Understanding the complete sequence of events during a single physics frame reveals how all these systems interact to produce the characteristic feel of Source Engine movement. The frame execution follows a carefully orchestrated sequence that balances gravity, input, collision, and velocity modifications.

### Frame Structure

Each server tick (frame) begins with pre-processing steps that prepare the player state for movement calculations. The `ProcessMovement` function serves as the entry point, determining which movement mode should execute based on the player's current state (ground, air, water, etc.).

For surfing, which occurs in the air movement state, the execution proceeds as follows:

**1. Gravity Application Start**

`StartGravity` executes first, applying half of this frame's gravitational acceleration. This adds negative Z velocity to the player, typically around 6 units at 66 tick (800 * 0.015 * 0.5 = 6). The player's velocity vector now points slightly more downward than it did at the end of the previous frame.

**2. Input Processing and Air Control**

The `AirMove` function calculates the player's intended movement direction (wishdir) based on view angles and key inputs. It determines wishspeed from input magnitudes and calls `AirAccelerate` to apply acceleration toward the wish direction. At this stage, the player gains control over their horizontal trajectory, able to steer along the ramp or adjust their approach to upcoming features.

**3. Collision Detection and Movement**

`TryPlayerMove` executes, attempting to move the player along their current velocity vector for the duration of this frame. The function traces the player's bounding box through the world, detecting any collisions. When the ramp surface is encountered, the trace returns collision information including the impact point and surface normal.

**4. Velocity Adjustment**

`ClipVelocity` processes the collision, removing the component of velocity directed into the ramp surface. The negative Z velocity from gravity (Step 1) and the horizontal velocity from air control (Step 2) are projected onto the ramp's angled surface. The perpendicular component is removed while the parallel component remains, effectively converting some downward velocity into forward velocity along the ramp.

**5. Ramp Bug Prevention (Momentum Mod)**

If using Momentum Mod, additional checks occur during the collision loop. The stuck-on-ramp detection examines whether the trace started inside solid geometry due to floating-point errors. If detected, the retrace logic activates, searching for a valid plane and backing the player out slightly. The vertical ramp bug check compares collision planes for parallelism, preventing the cross-product failure that causes stops on ramp seams.

**6. Multi-Collision Resolution**

If the player contacts multiple surfaces in a single frame (common on curved ramps or at geometry intersections), `TryPlayerMove` loops back to Step 3, processing each collision sequentially. The bump count limit (4 in vanilla, 8 in Momentum Mod) prevents infinite loops while allowing sufficient iterations to resolve complex collision scenarios.

**7. Ground State Determination**

`CategorizePosition` executes to determine if the player should be considered grounded or airborne for the next frame. Because the ramp's surface normal has a Z component less than 0.7, the player remains airborne. In Momentum Mod, predictive checks ensure the player isn't prematurely grounded when moving at high speeds or approaching edges.

**8. Gravity Application Completion**

`FinishGravity` applies the second half of gravitational acceleration, adding another approximately 6 units of negative Z velocity at 66 tick. This downward velocity is now ready to be converted into forward motion on the next frame when ClipVelocity processes it.

**9. Post-Processing**

Final checks ensure velocity remains within engine limits (typically 3500 units/second maximum), position is valid (not stuck in world geometry), and any touched triggers are processed (teleports, speed modifiers, etc.).

### Frame Timing and Consistency

The frametime value (gpGlobals->frametime) serves as the time scaling factor for all physics calculations. At 66 tick, this equals 0.015 seconds. At 100 tick, it equals 0.01 seconds. All acceleration, gravity, and velocity calculations scale by this value to ensure physics behavior remains consistent across different server tick rates.

Momentum Mod enforces fixed timesteps to prevent variance caused by server FPS fluctuations. In vanilla Source, if the server temporarily drops below its target tick rate, frametime increases for those frames, causing physics calculations to behave differently. This can lead to inconsistent jump heights, air acceleration rates, and surf physics. By locking frametime to a fixed value, Momentum ensures perfect reproducibility of movement sequences.

### The Complete Picture

Over the course of multiple frames, these systems combine to create the surfing experience. Gravity provides constant downward acceleration that serves as an energy source. ClipVelocity converts this vertical velocity into horizontal velocity along the ramp's surface. Air acceleration allows the player to control their trajectory and gain additional speed through strafing. Collision resolution handles complex geometry transitions without introducing random stops or stuck states.

The result is a physics system that rewards skill and understanding. Players who master view angle control, strafe timing, and ramp approach angles can chain together complex sequences of movements, achieving speeds and performing maneuvers impossible through normal gameplay mechanics. The emergence of surfing from these fundamental physics rules represents one of the most successful examples of emergent gameplay in video game history.

---

## Momentum Mod Improvements Summary

Momentum Mod's enhancements specifically target the randomness and inconsistency issues that plague vanilla Source surfing. Each improvement addresses a specific failure case in the original implementation while preserving the core physics that define Source Engine movement.

### Configuration Variables

**sv_ramp_bumpcount** (Default: 8)
- Increases collision resolution iterations from vanilla's hardcoded 4
- Prevents random stops on complex curved ramps
- Allows smooth surfing on geometry that would exhaust vanilla's bump limit
- Higher values provide more thorough collision resolution at minimal performance cost

**sv_ramp_fix** (Default: Enabled)
- Activates the retrace system for handling stuck-in-geometry states
- Detects when traces start inside solid due to floating-point precision errors
- Searches for valid exit planes and backs the player out smoothly
- Eliminates the most common cause of random mid-surf stops

**sv_slope_fix** (Default: Enabled)
- Preserves horizontal velocity when jumping onto inclined surfaces
- Prevents speed loss when boarding ramps from above
- Only activates when player has autobhop and is holding jump
- Creates smooth, consistent ramp entry without exploitable behavior

**sv_edge_fix** (Default: Enabled)
- Implements predictive ground detection to enable consistent edge bugs
- Prevents premature ground-snapping when approaching edges at speed
- Maintains proper ground detection for normal gameplay
- Allows skilled players to use edge techniques without pixel-perfect positioning

**sv_ramp_initial_retrace_length** (Default: 0.1 units)
- Controls how far the retrace system backs the player out when stuck
- Lower values minimize position correction but may not fully resolve stuck states
- Higher values ensure freedom from geometry but may cause noticeable position shifts
- Default value balances smooth movement with minimal visual artifacts

### Algorithmic Improvements

**Vertical Ramp Bug Fix**
The most elegant fix in Momentum Mod addresses the parallel plane collision issue with minimal code. When two collision planes are detected and found to be effectively identical (within floating-point tolerance), the code skips the problematic cross-product calculation. Instead, it pushes the player slightly away from the plane in the plane's normal direction, preserving horizontal velocity components while preventing the stuck state. This fix operates transparently, requiring no configuration and producing no side effects.

**Ramp Stuck Retrace System**
The retrace system represents a more comprehensive approach to floating-point error handling. When a trace begins inside solid geometry, vanilla Source immediately stops the player. Momentum Mod instead flags this as a stuck state and activates recovery logic. The system searches a 3×3×3 grid of directions (27 total) to find a valid plane that leads out of the stuck position. Once found, the player's origin is adjusted backward along that plane by the retrace length. This recovery happens in a single frame and is imperceptible to the player, but it prevents the jarring stop that would occur in vanilla.

**Slope Velocity Preservation**
The slope fix modifies ClipVelocity's behavior when specific conditions are met: the player must have autobhop enabled and be holding jump, the surface must be walkable (normal.z >= 0.7), and the clipping operation must reduce horizontal speed. When all conditions are satisfied, the fix restores the original horizontal velocity components while zeroing vertical velocity. This creates natural, consistent behavior when boarding ramps while preventing exploitation in other contexts.

**Predictive Ground Categorization**
Traditional ground detection only examines the current frame, leading to premature ground-snapping in many scenarios. Momentum Mod's predictive approach simulates the next frame's position based on current velocity and examines whether the player would be grounded at that future position. This forward-looking categorization prevents friction application during valid bhop chains and surf approaches while maintaining proper ground detection when players actually intend to land.

### Performance Considerations

All Momentum Mod improvements are designed with performance in mind. The increased bump count from 4 to 8 doubles the maximum collision iterations but only in worst-case scenarios involving extremely complex geometry. Most frames still resolve in fewer iterations. The retrace system only activates when stuck states are detected, adding no overhead to normal movement. The parallel plane check is a simple floating-point comparison that executes in nanoseconds. The slope fix adds one additional velocity magnitude comparison to ClipVelocity calls.

In practice, these improvements have negligible performance impact even on older hardware. The vastly improved consistency and elimination of random stops provide significant benefits to player experience at minimal computational cost.

### Comparison Table

| Feature | Vanilla Source | Momentum Mod |
|---------|------------------------|--------------|
| **Bump Count** | Hardcoded to 4 iterations | Configurable via `sv_ramp_bumpcount` (default 8) |
| **Ramp Bug** | Player stops when trace starts in solid | Retrace system backs player out smoothly |
| **Vertical Ramp Bug** | Cross-product fails on parallel planes, causing stops | Detects parallel planes and pushes player away |
| **Uphill Slopes** | ClipVelocity reduces horizontal speed | Slope fix preserves horizontal velocity when jumping |
| **Stairs/Steps** | Can generate downward velocity on steps | Step fix prevents excessive negative Z velocity |
| **Ground Detection** | Snaps to ground within 2 units | Predictive check prevents premature grounding |
| **Edge Bugs** | Requires pixel-perfect positioning | Edge fix enables consistent edge techniques |
| **Frametime** | Can vary with server FPS | Enforced fixed timestep for reproducibility |
| **Configuration** | No movement-related ConVars | Multiple ConVars for customization |
| **Debugging** | Limited visibility into failures | Extensive logging and visualization options |

---

## Conclusion

Surfing in the Source Engine represents a masterclass in emergent gameplay, where simple mathematical rules combine to create complex, skill-based movement. The fundamental mechanics-gravity as constant acceleration, ClipVelocity as vector projection, air acceleration as directional speed limiting-are straightforward individually but produce rich interaction when combined.

The vanilla Source SDK implementation provides the foundation but suffers from limitations appropriate to its era: hardcoded parameters, insufficient floating-point error handling, and assumptions about geometry complexity that don't hold for modern custom maps. Players adapted by learning to work around these limitations, developing elaborate techniques to avoid triggering bugs.

Momentum Mod's approach exemplifies modern game development philosophy: preserve the core mechanics that define the experience while eliminating randomness and inconsistency that frustrate players. Each fix targets a specific failure case with minimal, surgical changes to the codebase. The result is a movement system that feels identical to vanilla in spirit but dramatically more reliable in practice.

For developers, this analysis demonstrates the importance of understanding systems at a fundamental level. Surfing wasn't designed-it emerged from collision resolution math. But by understanding that math deeply, we can identify why failures occur and craft solutions that maintain the emergent properties while eliminating the bugs.

For players, this knowledge provides insight into what happens beneath the surface during every frame of movement. Understanding why looking sideways produces acceleration, why gravity enables speed gain on ramps, and why certain slopes feel "sticky" transforms surfing from mysterious black magic into a comprehensible, masterable skill.

The Source Engine's movement system has influenced game design for nearly two decades, spawning entire game modes, communities, and competitive scenes. Its continued relevance stems from the depth that emerges from relatively simple rules-depth that rewards mastery while remaining accessible to newcomers. Momentum Mod ensures this legacy continues with the reliability and polish that modern players expect.



