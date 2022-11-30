FROM confluentinc/cp-server-connect:7.3.0
USER root
RUN confluent-hub install --no-prompt jcustenborder/kafka-connect-spooldir:2.0.65 \
  && confluent-hub install --no-prompt confluentinc/kafka-connect-s3:10.3.0
USER 1001
