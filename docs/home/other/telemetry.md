# Telemetry

Genestack collects telemetry data to improve service quality, ensure stability, and enhance security. Analysis of both technical parameters such as resource utilization, and product metrics like anonymized feature usage data helps to improve ODM trustworthy for the most crucial aspects of it's work.

More specifically, telemetry helps to:

- detect and resolve issues more efficiently

- optimize system performance and scalability

- make informed decisions about product improvements

- enhance reliability and user experience.

Genestack doesn't collect any personal information without user consent, complying with data protection regulations; refer to security documents if you need more information in this regard.

Technical metrics includes groups of metrics to control:

- Hardware parameters, such as CPU and memory consumption, etc. For example:

    - `system_cpu_count` - The number of processors available to the Java virtual machine

    - `system_load_average_1m` - The sum of the number of runnable entities queued to available processors and the number of runnable entities running on the available processors averaged over a period of time

    - `process_cpu_usage` - The "recent cpu usage" for the Java Virtual Machine process

    - `process_uptime_seconds` - The uptime of the Java virtual machine

- Programming language-specific metrics to track JVM behaviour controlling heap size and other parameters. For example:

    - `jvm_memory_used_bytes` - The amount of used memory

    - `jvm_memory_max_bytes` - The maximum amount of memory in bytes that can be used for memory management

    - `jvm_threads_states_threads` - The current number of threads

    - `hikaricp_connections` - Total connections

    - `jdbc_connections_active` -  Current number of active connections that have been allocated from the data source.

In addition to technical metrics, anonymized product usage data is being collected. It helps to understand better which features are used and to guide product development decisions. These metrics help us identify what brings value to users and where improvements are needed. The list of collected metrics includes but not limited to:

- `product_number_of_active_users` - Number of registered unique not-disabled users

- `product_number_of_attached_files` - Number of Study File attachments by Data class

- `product_number_of_signal_groups` - Number of Tabular, Variants and Flow Cytometry groups

- `product_number_of_slp` - Number of Samples, Libraries, and Preparations

- `product_number_of_slp_groups` - Number of Sample, Library, and Preparation groups by Data class

- `product_number_of_studies` - Number of Studies

- `product_number_of_logged_in_users` - Number of logged in users

- `product_rest_api_requests_total` - Number of API calls grouped by application, path and method

- `product_user_session_duration` - User session duration
