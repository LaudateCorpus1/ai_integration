# ai_integration
[![PyPI version](https://badge.fury.io/py/ai-integration.svg)](https://badge.fury.io/py/ai-integration)
AI Model Integration for Python 2.7/3


# Built-In Integration Modes
There are several built-in modes for testing:

* Command Line using argparse (command_line)
* HTTP Web UI / multipart POST API using Flask (http)
* Pipe inputs dict as JSON (test_inputs_dict)
* Pipe inputs dict as pickle (test_inputs_pickled_dict)
* Pipe single image for models that take a single input named image (test_single_image)
* Test single image models with a built-in solid gray image (test_model_integration)

# Entrypoint Shims

Your docker entrypoint should be a simple python file (so small we call it a shim)
* imports start_loop from this library
* passes your inference function to it
* passes your inputs schema to it

The library handles everything else.


# Finished Model Container Requirements:

1. Working directory has your entrypoint shim in it. Set with WORKDIR

2. You install this library with pip (or pip3)

3. CMD is used in the your model dockerfile to specify the entrypoint as your shim.

4. No command line arguments will be passed to your entrypoint. (Unless using the command line interface mode)

5. To test your finished container's integration, run:
    * nvidia-docker run --rm -it -e MODE=test_model_integration YOUR_DOCKER_IMAGE_NAME
    * use docker instead of nvidia-docker if you aren't using NVIDIA...
    * You should see a bunch of happy messages. Any sad messages or exceptions indicate an error.
    * It will try inference a few times. If you don't see this happening, something is not integrated right.


# Inference Function Specification

inference_function is a function that takes a single argument:
- inputs: dict()
- keys are input names (typically image, or style, content)
- values are the data itself. Either byte array of JPEG data (for images) or text string.
- any model options are also passed here and may be strings or numbers. best to accept either strings/numbers in the model.

inference_function should return a dict():
```python
{
    'content-type': 'application/json', # or image/jpeg
    'data': "{JSON data or image data as byte buffer}",
    'success': True,
    'error': 'the error message (only if failed)'
}   
```

# Error Handling

If there's an error that you can catch:
- set content-type to text/plain
- set success to False
- set data to None
- set error to the best description of the error (perhaps the output of traceback.format_exc())

inference_function should never intentionally throw exceptions.
- If an error occurs, set success to false and set the error field.
- If your inference function throws an Exception, the library will assume it is a bad issue and restart the script, so that the framework, CUDA, and everything else can reinitialize.

# Inputs Schema

An inputs schema is a simple python dict {} that documents the inputs required by your inference function.

Not every integration mode looks at the inputs schema - think of it as a hint for telling the mode what data it needs to provide your function.

All mentioned inputs are assumed required by default.

The keys are names, the values specify properties of the input.

### Schema Data Types
- image
- text
- Suggest other types to add to the specification!

### Schema Examples

##### Single Image
By convention, name your input "image" if you accept a single image input
```python
{
    "image": {
        "type": "image"
    }
}
```

##### Multi-Image
For example, imagine a style transfer model that needs two input images.
```python
{
    "style": {
        "type": "image"
    },
    "content": {
        "type": "image"
    },    
}
```

##### Text
```python
{
    "sentence": {
        "type": "text"
    }
}
```

# Creating Integration Modes

A mode is a function that lives in a file in the modes folder of this library.


To create a new mode:

1. Add a python file in this folder
2. Add a python function to your file that takes two args:
    
    def http(inference_function=None, inputs_schema=None):
3. Attach a hint to your function
4. At the end of the file, declare the modes from your file (each python file could export multiple modes), for example:
```python
MODULE_MODES = {
    'http': http
}

```

Your mode will be called with the inference function and inference schema, the rest is up to you!

The sky is the limit, you can integrate with pretty much anything.

See existing modes for examples.
