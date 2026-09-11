# Vulkan Shader Profiler

`vulkan-shader-profiler` is a perfetto-based Vulkan shader profiler using the layering capability of the [Vulkan-Loader](https://github.com/KhronosGroup/Vulkan-Loader).

It allows you to visualize Vulkan application execution using Perfetto, providing detailed information about compute shaders to help identify performance bottlenecks, and associates Vulkan SPIR-V source code with the trace events.

Using the `vulkan-shader-profiler-extractor` and `vulkan-shader-profiler-runner`, it is also possible to extract a specific dispatch from the trace (using the `dispatchId` debug information from the trace), and replay it with the runner.

# Legal

`vulkan-shader-profiler` is licensed under the terms of the [Apache 2.0 license](LICENSE).

This is not an officially supported Google product. This project is not eligible for the [Google Open Source Software Vulnerability Rewards Program](https://bughunters.google.com/open-source-security).

---

## Dependencies

*   [Vulkan-Loader](https://github.com/KhronosGroup/Vulkan-Loader)
*   [Vulkan-Headers](https://github.com/KhronosGroup/Vulkan-Headers)
*   [SPIRV-Headers](https://github.com/KhronosGroup/SPIRV-Headers)
*   [SPIRV-Tools](https://github.com/KhronosGroup/SPIRV-Tools)
*   [perfetto](https://github.com/google/perfetto)
*   A working Vulkan implementation/driver.

---

## Building

`vulkan-shader-profiler` uses CMake for its build system.

### Host / ChromeOS / Linux

To compile it, please run:
```bash
cmake -B <build_dir> -S . \
  -DPERFETTO_SDK_PATH=<path-to-perfetto-sdk> \
  -DPERFETTO_TRACE_PROCESSOR_LIB=<path-to-libtrace_processor.a> \
  -DPERFETTO_INTERNAL_INCLUDE_PATH=<path-to-perfetto-include> \
  -DSPIRV_TOOLS_SOURCE_PATH=<path-to-spirv-tools-source-dir> \
  -DSPIRV_TOOLS_BUILD_PATH=<path-to-spirv-tools-build-dir>
cmake --build <build_dir>
```

For real-life examples, refer to:
*   ChromeOS [ebuild](https://chromium.googlesource.com/chromiumos/overlays/chromiumos-overlay/+/main/dev-libs/vulkan-shader-profiler/vulkan-shader-profiler-0.0.1.ebuild)
*   GitHub presubmit [configuration](.github/workflows/presubmit.yml)

#### Build Options

*   **Required:**
    *   `PERFETTO_SDK_PATH`: Path to [perfetto](https://github.com/google/perfetto) SDK (looks for `perfetto.cc` and `perfetto.h` in this directory).
    *   `PERFETTO_TRACE_PROCESSOR_LIB`: Path to `libtrace_processor.a` produced by a Perfetto build.
    *   `PERFETTO_INTERNAL_INCLUDE_PATH`: Path to Perfetto internal include directory (`<perfetto>/include`).
    *   `SPIRV_TOOLS_SOURCE_PATH`: Path to [SPIRV-Tools](https://github.com/KhronosGroup/SPIRV-Tools) source directory.
    *   `SPIRV_TOOLS_BUILD_PATH`: Path to where [SPIRV-Tools](https://github.com/KhronosGroup/SPIRV-Tools) is built.
*   **Optional:**
    *   `PERFETTO_LIBRARY`: Name of a Perfetto library already available (avoids compiling `perfetto.cc`).
    *   `BACKEND`: Perfetto backend to use:
        *   `InProcess` (default): The application generates the traces. Build options and environment variables control trace size and destination.
        *   `System`: Uses the system's `traced` daemon.
    *   `TRACE_MAX_SIZE` (InProcess only): Max trace size in KB. Can be overridden at runtime via `VKSP_TRACE_MAX_SIZE` (Default: `1024`).
    *   `TRACE_DEST` (InProcess only): Default trace output file. Can be overridden at runtime via `VKSP_TRACE_DEST` (Default: `vulkan-shader-profiler.trace`).

### Android

For Android, only the **Layer** and the **Runner** are supported for device execution.

1.  Clone the repository into your AOSP tree under `external/vulkan-shader-profiler`.
2.  Compile using Soong:
    ```bash
    m libVkLayer_shader_profiler vulkan-shader-profiler-runner
    ```

---

## Running an Application with Vulkan Shader Profiler

### On Linux

To run an application with the profiler layer enabled, ensure the following:

1.  The `Vulkan-Loader` can find the manifest in `manifest/vulkan-shader-profiler.json`. Set this using:
    ```bash
    export VK_ADD_LAYER_PATH=<path-to-vulkan-shader-profiler-manifest-dir>
    ```
2.  Enable the layer:
    ```bash
    export VK_LOADER_LAYERS_ENABLE="VK_LAYER_SHADER_PROFILER"
    ```

You can also extract memory contents of buffers and images used by a specific dispatch. This requires a first run to generate the trace, followed by a second run with `VKSP_EXTRACT_BUFFERS_FROM=<trace.spvasm>` set. This generates a `<trace.spvasm.buffers>` file for the runner. Buffers can also be extracted individually by setting `VKSP_EXTRACT_MULTIPLE_BUFFERS=1` and merged using `vulkan-shader-profiler-merge-buffers` (see `test/test-buffers.sh` for an example).

### On ChromeOS

Make sure you have emerged and deployed the `vulkan-shader-profiler`. Then run the application using `vulkan-shader-profiler.sh`, which sets up the necessary environment variables.

### On Android

1.  Push the library to the device:
    ```bash
    adb push $OUT/system/lib64/libVkLayer_shader_profiler.so /data/local/debug/vulkan/
    ```
2.  Enable the layer:
    ```bash
    adb shell setprop debug.vulkan.layers VK_LAYER_SHADER_PROFILER
    ```

### Using the Trace

Once traces are generated, you can view them using the [Perfetto trace viewer](https://ui.perfetto.dev).

---

## Extracting a Dispatch from a Trace

You can extract a single dispatch from a generated trace using the `dispatchId` found in the trace:

```bash
vulkan-shader-profiler-extractor -i <input_trace> -o <output_file> -d <dispatchId> [OPTIONS]
```

*   `-i`: Path to the trace generated by the layer.
*   `-o`: Output path (readable SPIR-V text by default).
*   `-d`: The `dispatchId` to extract.
*   `-b`: Output binary SPIR-V instead of text.
*   `-s`: Path to a shader file to use instead of the trace (see [Large Shaders](#large-shaders)).
*   `-v`: Enable verbose debug mode.

---

## Replaying a Shader with the Runner

Only programs extracted with `vulkan-shader-profiler-extractor` can be run with the runner:

```bash
vulkan-shader-profiler-runner -i <input> [OPTIONS]
```

*   `-i`: Path to the extracted SPIR-V program.
*   `-b`: Path to the associated buffers file (generated via `VKSP_EXTRACT_BUFFERS_FROM`).
*   `-c`: Disable counters to run without overhead.
*   `-e`: Target `spv_target_env` for text input conversion (default: `vulkan1.3`).
*   `-n`: Number of hot runs.
*   `-m`: Number of cold runs before benchmarking.
*   `-o`: Descriptor set index and binding of a buffer to dump (e.g., `1.2`).
*   `-p`: Force Vulkan queue global priority (0: low, 1: medium, 2: high, 3: realtime).
*   `-v`: Enable verbose debug mode.

### Using Counters in SPIR-V

You can profile specific sections of the program by adding non-semantic instructions:

```assembly
%vksp = OpExtInstImport "NonSemantic.VkspReflection.4"
...
%ct = OpExtInst %void %vksp StartCounter "my_section"
...
%un = OpExtInst %void %vksp StopCounter %ct
```

The runner will output the percentage of time spent in that section.

---

## How the Vulkan Shader Profiler Layer Works

`vulkan-shader-profiler` intercepts key Vulkan APIs to track execution and resources.

### Intercepted Calls for Tracing

Every intercepted call also generates a trace event for the function itself.

*   `vkGetDeviceQueue`: Creates internal structures to trace everything executed on this queue.
*   `vkAllocateCommandBuffers`: Creates internal structures for the command buffer.
*   `vkFreeCommandBuffers`: Cleans up internal structures for the command buffer.
*   `vkBeginCommandBuffer`: Initializes internal structures for command buffer recording.
*   `vkQueueSubmit`: Modifies submit information to add a timeline semaphore for tracking submission completion, and spawns a background thread to process completed query results.
*   `vkCmdDispatch`: Records dispatch details to associate with the trace event when executed.
*   `vkCmdBindPipeline`: Tracks the active pipeline for the command buffer.
*   `vkCreateComputePipelines`: Associates the pipeline with its compute shader stage.
*   `vkCreateShaderModule`: Disassembles the SPIR-V shader and writes it into the Perfetto trace.

### Intercepted Calls for Buffer Extraction

These calls are tracked to capture resource state for the extractor:

*   `vkUpdateDescriptorSets`
*   `vkCmdBindDescriptorSets`
*   `vkCmdPushConstants`
*   `vkAllocateMemory`
*   `vkCreateBuffer`
*   `vkBindBufferMemory`
*   `vkCreateImage`
*   `vkCreateImageView`
*   `vkBindImageMemory`
*   `vkCreateSampler`

### Vulkan APIs Used Internally

The layer calls these APIs internally to perform timing and buffer extraction:

*   **Timing & Synchronization:**
    *   `vkCreateSemaphore`, `vkDestroySemaphore`, `vkWaitSemaphores`: Used to track workload completion on the GPU.
    *   `vkCreateQueryPool`, `vkDestroyQueryPool`, `vkGetQueryPoolResults`, `vkCmdResetQueryPool`: Used to allocate and retrieve timestamps.
    *   `vkCmdWriteTimestamp`: Injected into command buffers to mark start/end times.
    *   `vkGetCalibratedTimestampsEXT`: Used to align GPU timestamps with the host CPU timeline.
    *   `vkGetPhysicalDeviceProperties`: Used to retrieve `timestampPeriod` to convert ticks to nanoseconds.
*   **Buffer/Image Extraction:**
    *   `vkCmdPipelineBarrier`, `vkCmdCopyBuffer`, `vkCmdCopyImage`: Used to copy resource data to host-visible staging memory.
    *   `vkMapMemory`, `vkUnmapMemory`: Used to read staging memory on the host.
    *   `vkGetImageMemoryRequirements`, `vkGetBufferMemoryRequirements`: Used to allocate appropriate staging memory.
    *   `vkDestroyImage`, `vkDestroyBuffer`, `vkFreeMemory`: Used to clean up staging resources.
    *   `vkGetPhysicalDeviceMemoryProperties`: Used to find suitable memory types for staging resources.

---

## Known Issues

### Large Shaders
Large shader code might get truncated or missing in Perfetto traces.

**Workaround:**
1.  Run with `VKSP_SHADER_DIR=<path>` set to an existing directory. The layer will dump binary `.spv` files there.
2.  Use the `-s` option with `vulkan-shader-profiler-extractor` to specify the dumped shader file instead of retrieving it from the trace.
