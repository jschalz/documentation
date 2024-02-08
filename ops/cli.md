# CLI

## Example Usage

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
