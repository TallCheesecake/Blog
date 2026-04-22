+++
title = "Vulkan WK1"
date = "2026-04-XX"
description = "This is a dev log of me working with vulkan with the vulkan rust bindings vulkanalia"
[taxonomies]
tags=["rust", "vulkan", "graphics"]
+++

# Intro
So far I've done the set up section of: <a href="https://kylemayes.github.io/vulkanalia/" target="_blank"
rel="noopener noreferrer">Vulkan Tutorial (Rust)</a>, this includes the basic window/event loop with
winit, Instance creation, Validation Layers, Physical Devices, Queues, and Logical Devices.

I'm going to do my best to summarize and explain each part of the code this is mainly acting as
a personal recap. Remmeber that I will get a lot of things wrong and this is more so for me
to review the thoughts than to actually provide any usefull information.

## Setup
We are using `pretty_loger`, `anyhow`, and `winit`. The first two are simple enough, just a logger and a 
error augmentation crate, the last one is a window library. `Winit`, this is a cross platform window 
library, its pretty standard with a window object having handles you can match on such as DeviceEvent
or WindowEvent. The run method on the eventloop obj continuosly polls, you pass it a closure and 
can match on the event emmited.

We then make the structs App and AppData. The latter will be the mass on structs used to paramaterize vulkan
and the former will be the render, creation, and drop logic for the vulkan instance.

## Instance
This is the first real vulkan think we will do, pretty much all of vulkan configuration is done through
structs (YAY!). Vulkanalia uses the builder pattern, we will pass a bunch of data into the struct like this:
```rust, linenos
unsafe fn create_instance(window: &Window, entry: &Entry) -> Result<Instance> {
    let application_info = vk::ApplicationInfo::builder()
        .application_name(b"Vulkan Tutorial\0")
        .application_version(vk::make_version(1, 0, 0))
        .engine_name(b"No Engine\0")
        .engine_version(vk::make_version(1, 0, 0))
        .api_version(vk::make_version(1, 0, 0));
}
```

Next we do the extentions, extrnetions are a way to "add on" features into the normal vulkan structs.
The window obj requires certain extentions and vulkanalia conveniently integrates with this.
The vk_window::get_required_instance_extentions() funciton returns a 'static slice of 
extention names, this will then be cast to c style strings with the .as_ptr() method.
This seems insane to me, to not really have a dedicated type for this but whatever.

With the pointer array of extentions and the app_info we can create a Instance.

{{ note( header="Note", body='Instance is a vulkanalia custom version of vk_instance, further note that we will not be providing custom allocators 
for object creation') }}

Now we need a entry point, this will let us query info about the Instance.

The Vulkan implimentation on certain platforms is not always compliant with the spec. To account for this we need
to add flags to our instance indicating what device "categories" we would like to add to our instance. 
By default vk instance's don't enumerate devices that require portability layers, these flags 
tell methods like enumeratePhysicalDevices to list devices that may need portability layering. The flags
are extracted with something like:

```rust, linenos
let flags = if 
    cfg!(target_os = "macos") && 
    entry.version()? >= PORTABILITY_MACOS_VERSION
{
    info!("Enabling extensions for macOS portability.");
    extensions.push(vk::KHR_GET_PHYSICAL_DEVICE_PROPERTIES2_EXTENSION.name.as_ptr());
    extensions.push(vk::KHR_PORTABILITY_ENUMERATION_EXTENSION.name.as_ptr());
    vk::InstanceCreateFlags::ENUMERATE_PORTABILITY_KHR
} else {
    vk::InstanceCreateFlags::empty()
};

let info = vk::InstanceCreateInfo::builder()
    .application_info(&application_info)
    .enabled_extension_names(&extensions)
    .flags(flags);
```
## Validation Layers

On to validation/debuging :(

The vulkan api in writen in C and has like litteraly no error handling. Instead they use "layers" these are pieces of
software that can intercept any vulkan API call and tell you information about it. I have been told that these layers
are dynamicly loaded by the "loader" but I have no idea what that is and will probably explain it in another blog.

You can enable and disable layers with a json file or env vars.


```rust, linenos
let available_layers = entry
    .enumerate_instance_layer_properties()?
    .iter()
    .map(|l| l.layer_name)
    .collect::<HashSet<_>>();

if VALIDATION_ENABLED && !available_layers.contains(&VALIDATION_LAYER) {
    return Err(anyhow!("Validation layer requested but not supported."));
}

let layers = if VALIDATION_ENABLED {
    vec![VALIDATION_LAYER.as_ptr()]
} else {
    Vec::new()
};
```
The layer VK_LAYER_KHRONOS_validation contains a bunch of usefull debuging layers. Here we ask the entry, with is 
pretty much that loader thing metioned earlier (I think),  to give us the layer and we put them in a HSet. You can now add
the enabled layers to the info variable (Instance).

We now actually need to have loging set up. Layers are all well and good but we need to be able to actually see what we are doing.

the layer "emits"? something called DebugUtilsMessageSeverityFlagsEXT. It looks like this, so its just a 
bitmask over VkDebugUtilsMessageSeverityFlagBitsEXT.
bitflags! {
    /// <https://www.khronos.org/registry/vulkan/specs/latest/man/html/VkDebugUtilsMessageSeverityFlagsEXT.html>
    #[repr(transparent)]
    #[derive(Default)]
    pub struct DebugUtilsMessageSeverityFlagsEXT: Flags {
        const VERBOSE = 1;
        const INFO = 1 << 4;
        const WARNING = 1 << 8;
        const ERROR = 1 << 12;
    }
}
{{ note( header="Note", body="I'm wondering why they cant use a enum?") }}

```rust, linenos
extern "system" fn debug_callback(
    severity: vk::DebugUtilsMessageSeverityFlagsEXT,
    type_: vk::DebugUtilsMessageTypeFlagsEXT,
    data: *const vk::DebugUtilsMessengerCallbackDataEXT,
    _: *mut c_void,
) -> vk::Bool32 {
    let data = unsafe { *data };
    let message = unsafe { CStr::from_ptr(data.message) }.to_string_lossy();

    if severity >= vk::DebugUtilsMessageSeverityFlagsEXT::ERROR {
        error!("({:?}) {}", type_, message);
    } else if severity >= vk::DebugUtilsMessageSeverityFlagsEXT::WARNING {
        warn!("({:?}) {}", type_, message);
    } else if severity >= vk::DebugUtilsMessageSeverityFlagsEXT::INFO {
        debug!("({:?}) {}", type_, message);
    } else {
        trace!("({:?}) {}", type_, message);
    }

    vk::FALSE
}
```
Here we can just compare against the integer and log out the value from the data emited from the calback function.
We will store this layers messenger in the App struct and the vk::DebugUtilsMessagerEXT type will be constructed 
with the builder pattern with the call back as one of the arguments. When a event of interest occurs the message
will be given to the callback with will emmit to stdout.

Vulkan structs have this neat thing they do where though a pointer to another stuct they allow extending the functionality
of a struct through a chain of pointers without breaking backwards comatibility. You must iter through a chain of structs
and get all the info about them (this also means vulkan does not know the type of every struct in the chain).

so now we are going to extend the info variable with the debug_callback messenger we just made like this:

```rust, linenos
if VALIDATION_ENABLED {
    info = info.push_next(&mut debug_info);
}
```
running the code at this stage yeilds (this is just a snippet):

 DEBUG vulkan_ex1 > (GENERAL) Inserted device layer "VK_LAYER_KHRONOS_validation" (MYPATH_HEHEH/coding/c/vulkan/1.4.341.1/x86_64/lib/libVkLayer_khronos_validation.so)
 DEBUG vulkan_ex1 > (GENERAL) vkCreateDevice layer callstack setup to:
 DEBUG vulkan_ex1 > (GENERAL)    <Application>
 DEBUG vulkan_ex1 > (GENERAL)      ||
 DEBUG vulkan_ex1 > (GENERAL)    <Loader>
 DEBUG vulkan_ex1 > (GENERAL)      ||
 DEBUG vulkan_ex1 > (GENERAL)    VK_LAYER_KHRONOS_validation
 DEBUG vulkan_ex1 > (GENERAL)            Type: Explicit


## Physical Devices





