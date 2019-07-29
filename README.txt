Documentation Author: Niko Procopi 2019

This tutorial was designed for Visual Studio 2017 / 2019
If the solution does not compile, retarget the solution
to a different version of the Windows SDK. If you do not
have any version of the Windows SDK, it can be installed
from the Visual Studio Installer Tool

Welcome to the Asynchronous Compute Tutorial!
Prerequesites: Basic Compute Particles,
	Physics -> Deformable Body -> Mass Spring
	
In this tutorial, we use a compute shader to make calculations
that mimic the look of cloth. All cloths and fabrics have some 
sort of spring effect, because cloth itself can stretch, compress,
and swing around, just like a spring. This tutorial removes
the geometry shader, and only draws points. 

Asynchronous Compute allows the Compute shader and the graphics
pipeline to run at the same time, so the graphics pipeline does
not need to wait for the compute shader to finish

In main.cpp, we have the setup() function.
In setup, we create a vector with the position and velocity
of every particle that is in our system. When we create GPU
buffers for this data, we use GL_DYNAMIC_DRAW, to let the GPU
know that we want to change the data often.

Notice how there are four GPU buffers, two buffers for 
each vector (two for position, two for velocity).
One position buffer [0] is allocated to be large enough
for all the position data in the vector, then it is given 
the vector data. The other position buffer [1] is allocated
to be large enough for the vector data, but it is left empty.
The same applies to the velocity data. Here is why:

We want this compute system to be 100% GPU based, we don't want
data to constantly go back and forth from CPU to GPU and back
to CPU. 

When we draw our first frame, position[0] and velocity[0]
will be used as input, then the compute shader will write to 
position[1] and velocity[1] as output, then the shader program
will use position[1] and velocity[1] to render the frame.

Here's the magic, when this frame is done, the buffers swap.
position[1] and velocity[1] are used as input, and they will
write to position[0] and velocity[0] as output (overwriting
the original data). Then position[0] and velocity[0] are used
to render the next frame.

This continues over and over, allowing for the entire particle
system to run on the GPU, without the need of going back and 
forth to the CPU.

On line 306 in the main.cpp Update function, you can see
how the buffers swap, with an explanation in the comments
of how it works

The compute shader's buffer[0] will always read position,
The compute shader's buffer[1] will always write position,
The compute shader's buffer[2] will always read velocity,
The compute shader's buffer[3] will always write velocity,

We change which buffer is bound to which index in Update
in main.cpp (line 312-316)

Keep in mind, the compute pipeline and the graphics pipeline
are sepearte, so when we the Vertex and Fragment shader
will not be in the same shader program as Compute.
Compute is in its own program, while Vertex and Fragment are in
another together. In Update(), we call glUseProgram to activate
the compute shader, then we use glDispatchCompute to run
the program on the GPU.

When we want to render our scene, we call glUseProgram to use
the graphics program (vertex and fragment shaders). With this
program, we use glDrawArrays to draw a series of points (GL_POINTS),
and we set the size of the points with glPointSize, which we set
to 4.0f, that means that each point will be 4 pixels wide, and 
4 pixels tall. We pass positionBuffer[0] to the graphics program,
which is the buffer that the compute program is READING from.

The compute shader program and the graphics shader program are 
running in simultaniously on the GPU, which fully optimizes the
power that a graphics card has to offer. We make the graphics pipeline
read the buffer that the compute pipeline is reading from, because
then there is no risk.

For example, if we had the graphics pipeline read the buffer
that compute is writing to (which we are NOT doing), then
the two programs can interfere with each other, and cause corruption.

[VertexShader.glsl]
The veretx shader simply takes in a point, and multiplies it
by the view and projection matrices to get screen position

[FragmentShader.glsl]
By default, we set our OpenGL clear color to white, 
so we render black pixels wherever the particles are.
To do this, we just write out_color = vec4(0);

[ComputeShader.glsl]
In this program, the position and velocity of each particle
is calculated. We have a 2D array of particles in our system.
When we called glDispatchCompute, we passed a parameter for 
number of X particles (row), and number of Y particles (col).
gl_GlobalInvocationID.x is the row ( 0 to nParticles.x )
gl_GlobalInvocationID.y is the row ( 0 to nParticles.y )

if (ID.x > 0), which means if we don't have the farthest-left
particle, then calculate force towards the particle to the 
left of the current particle

if (ID.x < nParticles.x), which means if we don't have the 
farthest-right particle, then calculate force towards the 
particle to the right of the current particle

Then do the same checks and calculations for vertically-linked
particles. When we're done calculating force, we take time
into consideration (which is given to the shader as a uniform)
to finally write the output buffer
	PositionOut[idx].xyz = p + (v * deltaT) + (0.5f * a * deltaT * deltaT);	
	VelocityOut[idx].xyz = vec3(v + a*deltaT);
	
Congratulations, you've maximized efficiency of your GPU
as much as it can possibly be maximized to this day. You
can draw graphics with the GPU cores that are 
designed/reserved for graphics, while also using GPU 
cores that are designed/reserved for compute. 

For most GPUs, that gives you the most utilization you can
possibly get, unless you are on an Nvidia RTX card, then
you have some cores that are designed / reserved for ray
tracing.	

How to improve:
Draw polygons, textures, and lights, to make the cloth
look like a physical object, rather than a series of points.
Try using this for other particle effects: explosions, water, etc.
