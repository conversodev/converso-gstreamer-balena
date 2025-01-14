# gstreamer-playground

Balena-based gstreamer debug and profiling setup

`gstreamer` is incredibly powerful but notoriously hard to learn, use and debug.

This project aim is to help debug and profile gstreamer pipeline.

Note that it will install gstreamer from the ubuntu repository.

## TL;DR

[![balena deploy button](https://www.balena.io/deploy.svg)](https://dashboard.balena-cloud.com/deploy?repoUrl=https://github.com/aethernet/gstreamer-playground)

1. Deploy to your balena device
2. Set the ENV vars for your pipeline
3. Point your browser to your balena device url or ip to get the result

| Var                       | Type                              | Default                   | Description                     |
| ------------------------- | --------------------------------- | ------------------------- | ------------------------------- |
| `PIPELINE`                | `string`                          | `videotestsrc ! fakesink` | Pipeline to execute             |
| `STOP_PIPELINE_AFTER_SEC` | `int` (seconds)                   | `10`                      | Stop the pipline after duration |
| `GST_TRACERS`             | `string` (tracers separated by ;) | `cpuusage;framerate`      | Gst-Shark tracers to use        |

(cf all env, at the end of this readme)

## Architecture

This currently should runs on AMD64 (tested) and Raspberry pi (ARMv7 and aarch64, untested).

It could run on a jetson (aarch64) but without hardware accelleration (todo)

## What's inside ?

- `ubuntu`
- `gstreamer`
- `gstreamer-plugins-good`
- `gstreamer-plugins-bad`
- `gstreamer-plugins-ugly`
- `gst-shark`
- `graphviz`
- `v4l-utils`
- `mkvtoolnix`
- `alsa`
- `xserver` ([balenablocks/xserver](https://hub.balena.io/balenablocks/xserver))
- `files` ([edwin3/files](https://hub.balena.io/edwin3/files))

## Running a pipe using `gst-launch`

Connect using `ssh` (either thru balena cli or cloud) into the service.

Use `gst-launch-1.0 -ve YOUR ! TEST ! PIPE`. Let it run as long as you need, press `CTRL+C` to interupt it.

### Using ENV

If you set `PIPELINE` env to a pipe, it will be run at startup.

ie : `PIPELINE=videotestsrc ! fakesink`

You can limit the running duration of the pipeline with `STOP_PIPELINE_AFTER_SEC` (in sec).

## Log level

Log level is set to 7 by default.
You can change it to any level using the `GST_DEBUG` environment variable (in docker-compose.yml, balena cloud, or in the running container).

Valid Levels are 0 to 9, check the [`gstreamer` doc](https://gstreamer.freedesktop.org/documentation/tutorials/basic/debugging-tools.html?gi-language=c) for more infos.

## Visualizing a pipeline

Thanks to `GST_DEBUG_DUMP_DOT_DIR` env var set in docker-compose.yml, `gstreamer` will output a `dot` file at each change of pipeline status. (PREROLL, RUNNING, ...).

Dots files will be automatically turned into `PDF` and put into the `/files/` folder.

You can retrieve them using the web interface (cf Acessing files below)

Note that files will be erase at startup, unless you use the `PERSIST_FILES` ENV.

If you prefer `png` over pdf, set `DOT_PROCESSING_PNG` to true.

### Manually turn the dots into images

Turn off the automatic process by adding `NO_DOT_PROCESSING` to the env.

The `dot` files are stored in `/files/dots` which are served on port 80 by `edwin3/files` block.

You can turn those to `png` or `pdf` with the following command :

`dot -Tpng /files/dots/DOT_FILE_OF_INTEREST.dot > /files/pipeline.png`

or

`dot -Tpdf /files/dots/DOT_FILE_OF_INTEREST.dot > /files/pipeline.pdf`

## Accessing files

You can access the files (dot, png, pdf, ...) using the files webserver.

Everything you put inside `/files` will be available at `http://ip_of_your_device` (or `http://short_hash_of_your_device.local`)

You can also activate the `public url` in balenacloud and access those thru there.

## Using GSTShark

(this is currently broken)

`Gstshark` is a profiling tool for your `gstreamer` pipeline. It's made by great folks at [https://ridgerun.com](Ridgerun).

[The doc of the tool is on their wiki](https://developer.ridgerun.com/wiki/index.php?title=GstShark), be sure to read it.

At the time of writing, GSTShark includes 9 tracers :

- `InterLatency` : Measures the latency time at different points in the pipeline.
- `ProcTime` : Measures the time an element takes to produce an output given the corresponding input.
- `Framerate` : Measures the amount of frames that go through a src pad every second.
- `ScheduleTime` : Measures the amount of time between two consecutive buffers in a sink pad.
- `CPUUsage` : Measures the CPU usage every second. In multiprocessor systems this measurements are presented per core.
- `Graphic` : Records a graphical representation of the current pipeline.
- `Bitrate` : Measures the current stream bitrate in bits per second.
- `Queue` Level : Measures the amount of data queued in every queue element in the pipeline.
- `Buffer` : Prints information of every buffer that passes through every sink pad in the pipeline. This information contains PTS and DTS, duration, size, flags and even refcount.

To activate GSTShark, you need to set a few vars in the environment (again using docker-compose, balenacloud or directly on the device) :

- `GST_DEBUG="GST_TRACER:7"`
- `GST_TRACERS="cpuusage"`
- `GST_SHARK_LOCATION=/files/traces` (set by default)

You can choose multiple tracer at the same time if you like, just separate them with ';' (ie. `"queue;framerate"`)

There's other options you can set if needed, refer to the GSTShark doc for more infos.

Tracers accepts parameters using this synthax : `GST_TRACERS="tracer(parameter=value)"`
The list of paramters for each tracer is in [the official doc](https://developer.ridgerun.com/wiki/index.php/GstShark_-_Tracers)

### Parsing traces to generate graphs

To make sense of the traces files generated by GSTShark, the easiest way it to plot them on a graph.

To do so GSTShark comes with a dedicated tool called `gstshark-plot`.

The traces will be auto-converted to pdf a few seconds after the end of the pipeline and put in the /files folder (cf Accessing Files).

#### Manually processing traces

You can turn off the auto-processing of traces by setting the `NO_TRACE_PROCESSING` env.

It's available in `/gst-shark/scripts/graphics/gstshark-plot`.

Usage is : `cd /gst-shark/scripts/graphics/ && ./gstshark-plot /files/traces -s pdf -l outside`

You can change the output format to `png` by replacing `pdf`, and choose another place for the legends (`inside`, `outside` or `extern`).

## XServer

An X11 Server is launched thanks to [balenablocks/xserver](https://hub.balena.io/balenablocks/xserver).

The start script will wait for the server to be ready before processing something else.
You can turn off the wait by setting `NOWAIT_X` to true in the ENV.

## Other tools

- `Alsa` : for audio test, you can use `arecord` and `aplay` to test audio devices
- `v4l-utils` : for changing camera parameters or listing devices with `v4l2-ctl`
- `mkvtoolnix` : if you want to play around with `matroska` files

### Useful commands :

- `arecord -l` : list audio devices availble for recording
- `aplay -l` : list audio devices availble for playing
- `v4l2-ctl --list-devices` : list video devices

## ENV

| Var                       | Type                              | Default                   | Description                                                                                                   |
| ------------------------- | --------------------------------- | ------------------------- | ------------------------------------------------------------------------------------------------------------- |
| `PIPELINE`                | `string`                          | `videotestsrc ! fakesink` | Pipeline to execute                                                                                           |
| `STOP_PIPELINE_AFTER_SEC` | `int` (seconds)                   | `10`                      | Stop the pipline after duration                                                                               |
| `GST_TRACERS`             | `string` (tracers separated by ;) | `cpuusage;framerate`      | Gst-Shark tracers to use                                                                                      |
| `GST_DEBUG`               | `string`                          | `GST_TRACER:7`            | Log Level (7 to get the traces) with filters to only get GST_TRACER from Gst-Shark                            |
| `GST_DEBUG_DUMP_DOT_DIR`  | `string`                          | `/files/dots/`            | Directory to ouput the dot files                                                                              |
| `PERSIST_FILES`           | `boolean`                         | `undefined`               | If set, will not delete previous dots, traces and graphs on rerun; it will probably override some of them tho |
| `DOT_PROCESSING_PNG`      | `boolean`                         | `undefined`               | Output the graphical representation of the pipeline as a PNG instead of PDF                                   |
| `NO_DOT_PROCESSING`       | `boolean`                         | `undefined`               | Prevent the production of a graphical representation of the pipeline as PDF or PNG                            |
| `GST_SHARK_LOCATION`      | `string`                          | `/files/traces/`          | Directory to ouput the traces files from gst-shark                                                            |
| `NO_TRACE_PROCESSING`     | `boolean`                         | `undefined`               | Prevent the production of a graphical representation of the traces as PDF                                     |
| `NOWAIT_X`                | `boolean`                         | `undefined`               | Don't wait for a X Server to be ready before running the pipeline                                             |

# TODO and next steps for this project

- [ ] make a jetson version
- [ ] make a webpage with options to run an arbitrary pipe and get the resulting visualizations and graphs
- [ ] build a living library of useful pipelines
- [ ] push everything to balena hub
