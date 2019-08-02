# DOODS
Dedicated Outside Object Detection Service - Yes, it's a backronym, so what...

DOODS is a GRPC service that detects objects in images. It's designed to be run as a container, optionally remotely. 

## API
The API uses gRPC to communicate but it has a REST gateway built in for ease of use. It supports both a single call RPC and a streaming interface.
It supports very basic pre-shared key authentication if you wish to protect it. It also supports TLS encryption but is disabled by default.
It uses the content-type header to automatically determine if you are connecting in REST mode or GRPC mode. It listens on port 8080 by default.

### GRPC
The protobuf API definitations are in the `odrpc` directory. There are 3 endpoints. 

- GetDetectors - Get the list of configured detectors.
- Detect -  Detect objects in an image - Data should be passed as raw bytes
- DetectStream - Detect objects in a stream of images

### REST/JSON
* `GET /detectors` - Get the list of configured detectors
* `POST /detect` - Detect objects in an image

```
{
	"detector_name": "default",
	"data": "<base64 encoded image information>",
	"detect": {
		"*": 50
	}
}
```

The result is returned as:
```
{
    "id": "test",
    "detections": [
        {
            "x1": 0,
            "y1": 8,
            "x2": 512,
            "y2": 587,
            "label": "person",
            "confidence": 87.890625
        }
    ]
}
```

Perform a detection. Use the default detector. (If omitted, it will use the default)
The `data`, when using the REST interface is base64 encoded image data. DOODS can decode png, bmp and jpg. 
The `detect` object allows you to specify the list of objects to detect as defined in the labels file. You can give a min percentage match.
You can also use "*" which will match anything with a minimum percentage.

Example One Liner:
```
echo "{\"detector_name\":\"default\", \"detect\":{\"*\":60}, \"data\":\"`cat grace_hopper.png|base64 -w0`\"}" > /tmp/postdata.json && curl -d@/tmp/postdata.json -H "Content-Type: application/json" -X POST http://localhass.prozach.org:8080/detect
```

## Detectors
You should optimally pass image data in the requested size for the detector. If not, it will be automatically resized.
It can read BMP, PNG and JPG as well as PPM.

### TFLite
If you pass PPM image data in the right dimensions, it can be fed directly into tensorflow. This skips a couple steps for speed. 
You can also specify the `type` of `tflite-edgetpu` in the config and it will enable Coral EdgeTPU hardware acceleration. 
You must also provide it an appropriate EdgeTPU model file.

## Compiling
This is designed as a go module aware program and thus requires go 1.12 or better. It also relies heavily on CGO. The easiest way to compile it
is to use the Dockerfile which will build a functioning docker image. It's a little large. 

## Configuration
The configuration can be specified in a number of ways. By default you can create a json file and call it with the -c option
you can also specify environment variables that align with the config file values.

Example:
```json
{
	"logger": {
        "level": "debug"
	}
}
```
Can be set via an environment variable:
```
LOGGER_LEVEL=debug
```

### Options:
| Setting                        | Description                                                 | Default      |
|--------------------------------|-------------------------------------------------------------|--------------|
| logger.level                   | The default logging level                                   | "info"       |
| logger.encoding                | Logging format (console or json)                            | "console"    |
| logger.color                   | Enable color in console mode                                | true         |
| logger.disable_caller          | Hide the caller source file and line number                 | false        |
| logger.disable_stacktrace      | Hide a stacktrace on debug logs                             | true         |
| ---                            | ---                                                         | ---          |
| server.host                    | The host address to listen on (blank=all addresses)         | ""           |
| server.port                    | The port number to listen on                                | 8080         |
| server.tls                     | Enable https/tls                                            | false        |
| server.devcert                 | Generate a development cert                                 | false        |
| server.certfile                | The HTTPS/TLS server certificate                            | "server.crt" |
| server.keyfile                 | The HTTPS/TLS server key file                               | "server.key" |
| server.log_requests            | Log API requests                                            | true         |
| server.profiler_enabled        | Enable the profiler                                         | false        |
| server.profiler_path           | Where should the profiler be available                      | "/debug"     |
| ---                            | ---                                                         | ---          |
| pidfile                        | Write a pidfile (only if specified)                         | ""           |
| profiler.enabled               | Enable the debug pprof interface                            | "false"      |
| profiler.host                  | The profiler host address to listen on                      | ""           |
| profiler.port                  | The profiler port to listen on                              | "6060"       |
| ---                            | ---                                                         | ---          |
| doods.auth_key                 | A pre-shared auth key. Disabled if blank                    | ""           |
| doods.detectors                | The detector configurations                                 | <see below>  |

### TLS/HTTPS
You can enable https by setting the config option server.tls = true and pointing it to your keyfile and certfile.
To create a self-signed cert: `openssl req -new -newkey rsa:2048 -days 3650 -nodes -x509 -keyout server.key -out server.crt`

### Detector Config
Detector config must be done with a configuration file. The default config detector config is:
```
{
	"doods": {
		"config": [
			{
				"name": "default",
				"type": "tflite",
				"model_file": "model.tflite",
				"label_file": "labels.txt",
				"num_threads": 4,
				"num_interpreters": 4,
			},
			{
				"name": "edgetpu",
				"type": "tflite-edgetpu",
				"model_file": "model-edgetpu.tflite",
				"label_file": "labels.txt",
				"num_threads": 4,
				"num_interpreters": 4,
			}
		]
	}
}
```

The default model is downloaded from google: coco_ssd_mobilenet_v1_1.0_quant_2018_06_29
The default edgetpu model is mobilenet_ssd_v2_coco_quant_postprocess_edgetpu
If you do not have an edgetpu, this detector will not be enabled.

### Detectors
 * tflite - Tensorflow lite models
 * tflite-edgetpu - Tensorflow lite with Coral EdgeTPU hardware accelerator

## Examples
See the examples directory for sample clients

## Docker
To run the container in docker you need to map port 8080. If you want to update the models, you need to map model files and a config to use them. 
`docker run -it -p 8080:8080 snowzach/doods:latest`

### Coral EdgeTPU
If you want to run it in docker using the Coral EdgeTPU, you need to pass the device to the container
`docker run -it --device /dev/bus/usb -p 8080:8080 snowzach/doods:latest`

## Misc
Special thanks to https://github.com/mattn/go-tflite as I would have never been able to figure out all the CGO stuff. I really wanted to write this in Go but I'm not good enough at C++/CGO to do it. Most of the tflite code is taken from that repo and customized for this tool.