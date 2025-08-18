# Release notes

## Version 1.61

!!! tip ""
    Helm chart version 1.61.0

### Helm configuration changes

- Mailcatcher was replaced by Mailpit. The `mailcatcher` section has been removed, please use the `mailpit` configuration instead.

    From

      ```yaml
      mailcatcher:
        image:
          repository: dockage/mailcatcher
      ```

    To

      ```yaml
      mailpit:
        image:
          repository: axllent/mailpit
      ```
