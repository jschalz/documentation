# CLI

## External Processor

The Bulwark `ext-processor` subcommand launches an Envoy external processor service. It's currently the way that Bulwark handles and responds to incoming request traffic. The service requires the Envoy HTTP proxy server to be configured to use it. An [example configuration](https://github.com/bulwark-security/bulwark/blob/main/crates/ext-processor/examples/envoy.yaml) is available and can be adapted to a wide range of use cases.

### Example Usage

This will launch Bulwark as an Envoy external processor service, with a `debug` log level and [Elastic Common Schema](https://www.elastic.co/guide/en/ecs/current/ecs-reference.html) (ECS) formatted logs. It will read its [configuration](configuration.md) from a file named `main.toml` in the current working directory.

```
bulwark-cli --log-level debug --log-format ecs ext-processor --config=main.toml
```

### CLI Options

```
Bulwark is a fast, modern, open-source web application security engine.

Usage: bulwark-cli [OPTIONS] [COMMAND]

Commands:
  ext-processor  Launch as an Envoy external processor
  build          Compile a Bulwark plugin
  help           Print this message or the help of the given subcommand(s)

Options:
  -l, --log-level <LOG_LEVEL>
          Log levels: error, warn, info, debug, trace
          
          Default is "info".

  -f, --log-format <LOG_FORMAT>
          Log formats: ecs, forest
          
          Default is "ecs".

  -h, --help
          Print help (see a summary with '-h')

  -V, --version
          Print version
```

```
Launch as an Envoy external processor

Usage: bulwark-cli ext-processor --config <FILE>

Options:
  -c, --config <FILE>  Sets a custom config file
  -h, --help           Print help
```

## Build

The `build` subcommand [builds a plugin project](../api/building-plugins/) into a compiled WebAssembly binary that can be used by Bulwark. The output parameter can be either a filename or a directory. If supplied a directory name, it will use the plugin's name as the output filename within that directory.

```
Compile a Bulwark plugin

Usage: bulwark-cli build [OPTIONS] [-- <COMPILER_ARGS>...]

Arguments:
  [COMPILER_ARGS]...  Additional arguments passed through to the compiler

Options:
  -p, --path <FILE>    Sets the input directory for the build
  -o, --output <FILE>  Sets the output file for the build
  -h, --help           Print help (see more with '--help')
```

