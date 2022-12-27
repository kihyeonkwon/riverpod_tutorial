```
name: my_app_name
environment:
  sdk: ">=2.17.0 <3.0.0"
  flutter: ">=3.0.0"

dependencies:
  flutter:
    sdk: flutter
  flutter_riverpod: ^2.1.1
  riverpod_annotation: ^1.0.6

dev_dependencies:
  build_runner:
  riverpod_generator: ^1.0.6
```

Finally, run the code-generator with flutter pub run build_runner watch
