+++
title = "Vulkan WK1"
date = "2026-04-22"
description = "This is a dev log of me working with vulkan with the vulkan rust bindings vulkanalia"
[taxonomies]
tags=["rust", "vulkan", "graphics"]
+++

# Intro
So far I've done the setup section of: <a
href="https://kylemayes.github.io/vulkanalia/" target="_blank" rel="noopener
noreferrer">Vulkan Tutorial (Rust)</a>. This includes the basic window/event
loop with `winit`, instance creation, validation layers, physical devices,
queues, and logical devices.

I'm going to do my best to summarize and explain each part of the code. This is
mainly acting as a personal recap. Remember that I will get a lot of things
wrong and this is more so for me to review my thoughts than to actually provide
useful information.

## Setup
We are using pretty_logger, anyhow, and winit. The first two are simple
enough, a logger and an error augmentation crate. The last one is a window
library. winit is a cross-platform window library. It's pretty standard, with a
window object exposing handles you can match on, such as DeviceEvent or
WindowEvent.

The run method on the event loop continuously polls events. You pass it a
closure and can match on the events emitted.

We then define the structs App and AppData. The latter will hold most of the
Vulkan-related state/configuration data, and the former will handle render
logic, creation, and cleanup (drop logic) for the Vulkan instance.

## Instance
This is the first real Vulkan thing we will do. Pretty much all Vulkan
configuration is done through structs (yay!). Vulkanalia uses the builder
pattern, so we pass a bunch of data into structs like this:
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
Next we handle extensions. Extensions are a way to "add on" features to the
core Vulkan API. The window system requires certain extensions, and Vulkanalia
conveniently integrates with this.

The vk_window::get_required_instance_extensions() function returns a 'static
slice of extension names. These are then passed to Vulkan as C-style strings
via .as_ptr(). This feels a bit low-level, but it’s expected since Vulkan
itself is a C API.

With the extension pointer array and the application info, we can create an
instance.

{{ note(header="Note", body="Instance is Vulkanalia's wrapper around
vk::Instance. Also note that we are not providing custom allocators for object
creation.") }}

Now we need an entry point. This allows us to query Vulkan functionality from
the loader and access instance-level operations.

On certain platforms, Vulkan implementations are not fully conformant. To
account for this, we may need to enable additional flags when creating the
instance. For example, on macOS, portability extensions may be required so that
Vulkan can run on top of Metal via a compatibility layer.

By default, VK instances do not enumerate devices that require portability
layers. These flags allow functions like enumerate_physical_devices to include
such devices.

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
The layer VK_LAYER_KHRONOS_validation contains a bunch of useful debugging
layers. Here we ask the entry (which is pretty much the loader thing mentioned
earlier, I think) to give us the available layers and we put them into a
HashSet.

We can then add the enabled layers to the info variable when creating the
Instance.

At this point, we also need to set up logging. Layers are all well and good,
but we need a way to actually see what is going on.

The validation layer emits something called DebugUtilsMessageSeverityFlagsEXT.
This is essentially a bitmask over VkDebugUtilsMessageSeverityFlagBitsEXT.
``` rust, linenos
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
}```

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
Here we can just compare against the integer values and log out the data
emitted from the callback function.

We will store this layer’s messenger in the App struct, and the
vk::DebugUtilsMessengerEXT type will be constructed using the builder pattern,
with the callback passed as one of its arguments. When an event of interest
occurs, the message is passed to the callback, which will emit it to stdout.

Vulkan structs have this neat feature where, through a pointer to another
struct, they allow extending functionality via a chain of structures without
breaking backwards compatibility. You must iterate through this chain of
structs and extract all the relevant information. This also means Vulkan does
not necessarily know the type of every struct in the chain at compile time.

So now we are going to extend the info variable with the debug callback
messenger we just created, like this:

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

When we talk about physical devices we refer to the actual GPU in the computer. We will need to check the type and features of this GPU with:

```rust, linenos
unsafe fn check_physical_device(
    instance: &Instance,
    data: &AppData,
    physical_device: vk::PhysicalDevice,
) -> Result<()> {
    let properties = instance.get_physical_device_properties(physical_device);
    if properties.device_type != vk::PhysicalDeviceType::DISCRETE_GPU {
        return Err(anyhow!(SuitabilityError("Only discrete GPUs are supported.")));
    }

    let features = instance.get_physical_device_features(physical_device);
    if features.geometry_shader != vk::TRUE {
        return Err(anyhow!(SuitabilityError("Missing geometry shader support.")));
    }

    Ok(())
}
}
```

Queues are features supported by the device. Each queue can handle a set of
operations, such as memory transfers or compute tasks. The graphics queue (as
far as I understand) supports most or all available operations.

The get_physical_device_queue_family_properties function will give us
information about the available queue families for our device. From this, we
can derive a u32 value that represents the QueueFamilyIndices. We can then
select a suitable physical device based on our technical requirements
(dedicated GPU, etc.).

## Logical Devices

A logical device is Vulkan’s representation of a physical device. It contains
metadata and other information used for interacting with the actual hardware.

Like all the other objects used so far, this one is also configured using
structs. The first thing we do is create a queue info struct. This describes
the queues we want the logical device to expose. In our case, we want the
graphics queue.

Creating a logical device requires information about device-specific layers,
features, and the queues we want to enable. The creation follows the same
pattern as instance creation:

```rust, linenos
let info = vk::DeviceCreateInfo::builder()
    .queue_create_infos(queue_infos)
    .enabled_layer_names(&layers)
    .enabled_extension_names(&extensions)
    .enabled_features(&features);
```

## End of Setup
That’s the end of the setup section. I will be releasing my notes on the presentation section shortly.
