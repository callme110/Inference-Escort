# Adding a new scanner

Inference-Escort can be extended to support new scanners, and to support additional models for the existing. These scanners could be added **via code** or **ad-hoc as part of the request**.

## Extending the input (prompt) scanners

1. Create a new class in the `inference_escort/input_scanners` that inherits from `base.Scanner` and implements the `scan` method. The `scan` method should return a tuple `str, bool, float`.
2. Add test cases for the new scanner in `tests/input_scanners`.
3. Add the new scanner to the `inference_escort/input_scanners/__init__.py` `__all__` enum.
4. Write documentation in the `docs/input_scanners` folder and add a link to the `mkdocs.yml` file.
5. Also, add a link to the documentation in `README.md`, and update the `docs/changelog.md` file.

## Extending the output scanners

1. Create a new class in the `inference_escort/output_scanners` that inherits from `base.Scanner` and implements the `scan` method. The `scan` method should return a tuple `str, bool, float`.
2. Add test cases for the new scanner in `tests/output_scanners`.
3. Add the new scanner to the `inference_escort/output_scanners/__init__.py` `__all__` enum.
4. Write documentation in the `docs/output_scanners` folder and add a link to the `mkdocs.yml` file.
5. Also, add a link to the documentation in `README.md`, and update the `docs/changelog.md` file.
