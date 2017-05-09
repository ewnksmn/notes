
# How exactly a shader parameter goes from application to a specific execution of pipeline?

## Mental model of how CPU and GPU works together

It has to be made clear that the CPU and GPU always work in parallel, i.e. a draw call always returns instantly, no matter it is invoked on a immediate context or a deferred one.

To transfer anything from application to the GPU, one has to go through this: 

`application -> runtime(D3D, OpenGL, Vulkan, etc) -> driver -> GPU`

So when a draw call returns, it returns from the runtime, which implies it is not executed on the GPU yet. The driver will hold this draw call and its required render states until the GPU has time to process it. This is essentially an
async event system: application uses the runtime to issue event, while the driver maintains a event queue to hold incoming events(and their parameters), which are then consumed by GPU.

> Newer APIs like D3D12, Vulkan and Nintendo's NVN tend to reduce the complexity of runtime and transfer driver responsibilities/power into the hand of an application programmer.
	
> In a sense, the D3D11's concept of immediate context is an illusion. That's why it doesn't exist in newer APIs like NVN.

Under this text book producer/consumer model, it's easier to see the big picture:

- A draw call is like an event, or a function, or a function object
- Render states and resources are input to that function
- The various pipeline stages altogether define the function signature 

## Memory access patterns of render resources

The entire input of a draw call can be complex, but it is much simpler if viewed from the standing point of memory access.

- Constant input 
	+ input that doesn't change after initialization, 
	+ e.g. static buffers, static textures, state objects.

- Dynamic input 
	+ input that needs to be changed between draw calls
	+ e.g. cbuffer/ubo, dynamic buffers, dynamic textures

- Transitioned input 
	+ the input of one draw call comes from the result of another draw call 
	+ e.g. render to texture

> TODO Add details about resource management, driver's buffer renaming behavior, cache coherency, and UMA

## Constant buffer usage pattern

### Requirements
- Minimal update
	+ const buffers are updated due to per-frame and per-object frequencies

- Reuse
	+ Reuse an existing const buffer when a smaller one is needed

- Multi-threaded rendering support

- Flexible application code
	+ in HLSL one can define arbitrary const buffer layouts
	+ support `shader.setParam("name", value);` where the shader object is of a generalized type
	+ const buffer layout is not known at compile time, but constructed from shader reflection data at runtime
	+ reasonable performance

### Design	

#### UE4's constant buffer pool
##### Desc
A uniform parameter holds reference to a buffer. When it is bound to the pipeline, it binds the underlying buffer.  When it needs to be updated, it:
	- releases the referenced buffer back to the pool
	- requests another buffer from the pool 
	- update the requested buffer

The buffer pool can shrink: any buffers that are not used for 30 frames will be destroyed.

The buffer pool is essentially a hashmap, it maintains a array of buckets, each bucket holds a list of fixed-size constant buffers(the size is 2^n where n ~[4, 16])

##### Pros
- The uniform parameter can maintain a main memory cache to perform redundancy check and achieve minimal update.
- Reuse is done easily when requesting buffers from pool
- Multi-threaded rendering is well supported. Each thread context can operate on a standalone uniform parameter.

##### Cons
+ When the same uniform parameter needs to be updated for 100 draw calls, it actually requests and updates 100 constant buffers once for each draw call.
This effectively bypasses the driver's constant buffer renaming feature, which is faster. 

+ Updating one cbuffer 100 times is faster than updating 100 cbuffers(of the same size) once. Given the pool nature, the 100 cbuffers can be larger than required, making it slower.

#### Each shader manages its own set of cbuffers
##### Desc
Each shader manages its own set of cbuffers. So shader and cbuffer objects have a one-to-one relationship.

##### Pros
Good use of drivers' cbuffer renaming feature, good performance

##### Cons
+ No cbuffer reuse between shaders, waste of video memory when there are many shaders that uses similar cbuffer layouts

+ Not easy to support multi-threaded rendering: 
	- when multiple threads issue draw call with the same shader, they actually try to Map/UnMap the same cbuffer
	- when this happens, the Map/UnMap has to be synchronized, too bad.

> Different threads should operate on different cbuffers to avoid thread synchronization.

+ Not easy to support minimal update 
	- if one cbuffer has to be updated multiple times on the same thread, each update must be flushed immediately so that XXSetConstantBuffers() can properly record the underlying renamed buffers.
	- to perform redundancy check under this situation, we must create/manage multiple caches, one for each thread.

#### Use the same cbuffer layout for all shaders
##### Desc
Given a set of shaders, find the largest cbuffer's size for each binding slot, and create cbuffers accordingly.
Or put it another way, create a set of cbuffers that are large enough for any given shaders.

##### Pros
- Reuse is in the nature of this design
- Multi-threaded rendering is supported as each thread context can have its own set of cbuffers
- Well use of cbuffer renaming feature

##### Cons
- Minimal update becomes meaningless. When updating a cbuffer for a draw call, only a partial part of that update is needed by the shader.
- Cbuffer renaming would waste memory on unused part.

#### Make use of large cbuffer(D3D11.1 feature)
https://www.gamedev.net/topic/682089-how-to-use-a-big-constant-buffer-in-directx-111/?view=findpost&p=5310348

