# Plugin Spec Explorations
This is a place to discuss a potential new open-source audio plugin format with modern features and futureproofness.

# Objective
Why create a new plugin standard?

We believe current audio plugin standards don't fit the needs of modern plugin developers and artists.
Problems with existing solutions include:
* outdated technology with missing key features (VST, LV2, LADSPA)
* closed and/or proprietary ecosystems with licensing hurdles (VST, VST3, AAX, AU)
* not cross-platform (AAX, AU)

As a community of passionate developers and artists, we believe we are equipped to create a new standard from the ground-up to support the needs of modern plugin developers and artists, as well as leaving room for robust futureproofness. This document aims to spark discussions on what features should be implemented in the final spec, and how to best futureproof this spec.

# Feature Checklist
This checklist is currently incomplete. Please open a PR with your additions/removals!

## 1. Static metadata the host reads before opening
This system provides the host with useful metadata info without having to open the plugin itself. This should use the TOML format.

```toml
[about]
plugin = [
  { language = "english", name = "My Plugin" },
  # ..
]
author = [
  { language = "english", name = "Spicy Plugins & Co." },
  # ..
]
description = [
  { language = "english", desc = "Hot stuff" },
  # ..
]

[ports]
audio-outputs = 2
audio-inputs = 2
audio-sidechain-inputs = 0
# Signifies to the host that an input is directly connected to an output. This allows the
# host to use "process_replacing" if it wishes to. If the number of inputs does not equal
# the number of outputs, then the host will ignore this.
audio-through = true  
midi2-outputs = 0
midi2-inputs = 1

[category]
categories = ["effect", "gain", "utility"]

[misc]
num-parameters = 3
has-custom-ui = true
```

## 2. Plugin Model
The state of all the parameters in the plugin. Unlike other plugins formats, this uses no parameter IDs. Instead, the host owns this model and passes it to the plugin as a parameter. This is analogues to the [`baseplug`] project. In fact, this is designed to fit in nicely with [`baseplug`] itself.

While this spec uses Rust as its official language, the actual spec should be made to be compatible with C.

Each parameter in the static model is structered as follows:
```rust
struct Parameter {
  // (language, name, short name)
  name: Vec<(&'static str, &'static str, Option<&'static str>)>
  
  // (language, description)
  description: Option<Vec<(&'static str, &'static str)>>
  
  // (min, max) Optional labels to the host can put at the min/max ends of the parameter.
  end_labels: Option<(&'static str, &'static str)>  
  
  // Whether or not this parameter accepts sample-accurate automation.
  accepts_sample_accurate: bool,
  
  // A pointer to the function that will be called when a parameter is changed. The host must
  // ensure that buffer_len and the length of sample_accurate is the same.
  update: Fn(&mut Model, normalized: f32, buffer_len: usize, sample_accurate: Option<&[f32]>),
  
  // A pointer to the function that returns what value the host should display to the user
  // given the normalized value.
  display: Option<Fn(f32) -> &str>,
  
  // A pointer to the function that returns the normalized value of the parameter.
  // (useful for loading/saving presets)
  get_normalized: Fn(&Model) -> f32,
}
```

## 3. Plugin MIDI2 I/O
The plugin spec should use MIDI2, as it supports the modern features we want and it's backwards compatible with MIDI.

## 4. Plugin/Host Interface
Now for a list of all the methods that need to be implemented in a plugin for communication with the host:

```rust
trait Plugin {
  type Model;
  
  // Retrieve the list of parameters.
  fn parameters() -> Vec<Parameter>  // This is the Parameter struct from before described above.
  
  // Create a new instance of this plugin.
  fn new() -> Result<Self::Model, String>;
  
  // Close this plugin instance. Move the owned model back to the trait to be destroyed.
  fn close(model: Self::Model);
  
  // Open the window of any custom GUI. The plugin should do nothing if a window is already
  // opened for this instance.
  fn request_gui_open(model: &mut Self::Model) -> Result<(), String>;
  
  // Close the window of any custom GUI.
  fn request_gui_close(model: &mut Self::Model);
  
  // If specified to use audio-through (process replacing), the host may call this function
  // instead of `process()`. (but it doesn't have to)
  fn process_through(
    // The audio buffers from each audio-through port (process replacing). When a buffer is
    // `None`, it means that port is not connected.
    audio_through: Vec<Option<&mut[f32]>>,
    
    // The sidechain input buffers for each port. When a buffer is `None`, it means that
    // port is not connected.
    audio_sidechain: Vec<Option<&[f32]>>,
    
    // The host must ensure this matches the length of each buffer.
    n_frames: usize,
  ) -> Result<(), String>
  
  fn process(
    // The audio buffers from each input port. When a buffer is `None`, it means that
    // port is not connected.
    audio_inputs: Vec<Option<&[f32]>>,
    
    // The audio buffers from each input port. When a buffer is `None`, it means that
    // port is not connected.
    audio_outputs: Vec<Option<&mut[f32]>>,
    
    // The sidechain input buffers for each port. When a buffer is `None`, it means
    // that port is not connected.
    audio_sidechain: Vec<Option<&[f32]>>,
    
    // The host must ensure this matches the length of each buffer.
    n_frames: usize,
  ) -> Result<(), String>
  
  // TODO: MIDI2
}
```

[`baseplug`]: https://github.com/wrl/baseplug
