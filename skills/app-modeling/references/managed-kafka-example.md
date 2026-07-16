# Managed Kafka Example

Use this example only after the selected source, exact `Radius.Messaging/kafka` schema, and target Environment recipe prove the same contract. It demonstrates the default Azure Event Hubs recipe shape where:

- `properties.host` is the Event Hubs namespace name, not a bootstrap endpoint;
- the bootstrap endpoint is `<host>.servicebus.windows.net:9093`;
- the client uses `SASL_SSL` with the `PLAIN` mechanism and username `$ConnectionString`; and
- `properties.secrets.name` identifies a managed secret containing the `connectionString` key.

An unmodified Kafka client must receive every native setting it consumes. For a client that accepts Kafka properties through environment variables, bind the secret before the setting that expands it:

```bicep
resource kafka 'Radius.Messaging/kafka@2025-08-01-preview' = {
  name: 'kafka'
  properties: {
    environment: environment
    application: app.id
    topic: 'events'
  }
}

resource uiContainer 'Radius.Compute/containers@2025-08-01-preview' = {
  name: 'kafka-ui'
  properties: {
    environment: environment
    application: app.id
    containers: {
      kafkaUi: {
        image: '<exact-registry pinned image>'
        ports: {
          web: {
            containerPort: 8080
          }
        }
        env: {
          KAFKA_CLUSTERS_0_NAME: {
            value: 'event-hubs'
          }
          KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: {
            value: '${kafka.properties.host}.servicebus.windows.net:9093'
          }
          KAFKA_CLUSTERS_0_PROPERTIES_SECURITY_PROTOCOL: {
            value: 'SASL_SSL'
          }
          KAFKA_CLUSTERS_0_PROPERTIES_SASL_MECHANISM: {
            value: 'PLAIN'
          }
          RAD_SECRET_CONNECTIONSTRING: {
            valueFrom: {
              secretKeyRef: {
                secretName: kafka.properties.secrets.name
                key: 'connectionString'
              }
            }
          }
          KAFKA_CLUSTERS_0_PROPERTIES_SASL_JAAS_CONFIG: {
            value: 'org.apache.kafka.common.security.plain.PlainLoginModule required username="$ConnectionString" password="$(RAD_SECRET_CONNECTIONSTRING)";'
          }
        }
      }
    }
  }
}
```

The environment variable names above are application-specific, not Radius conventions. Confirm them from the pinned source. Kubernetes expands `$(RAD_SECRET_CONNECTIONSTRING)` only because the helper appears first in the container environment and the container recipe preserves environment order; verify both facts for the exact runtime.

Do not add a generic connection when the application does not consume its projected `CONNECTION_*` values and relationship metadata is not required. Do not add `Radius.Compute/routes` merely because the workload exposes HTTP, and do not enable optional dynamic configuration without its complete writable-storage contract.

If the target schema or recipe uses a different endpoint shape, protocol, auth mechanism, or secret key, this example does not apply. Resolve that contract instead of adapting names by analogy.
