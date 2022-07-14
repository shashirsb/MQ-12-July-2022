FROM openjdk:7

ENV ACTIVE_MQ_VERSION 5.10.0
ENV ACTIVE_MQ apache-activemq-$ACTIVE_MQ_VERSION

RUN wget -O $ACTIVE_MQ.tar.gz http://archive.apache.org/dist/activemq/$ACTIVE_MQ_VERSION/$ACTIVE_MQ-bin.tar.gz 

RUN tar -xzf $ACTIVE_MQ.tar.gz --directory=/opt/

EXPOSE 8161 5672

CMD ["/bin/bash", "-c", "/opt/$ACTIVE_MQ/bin/activemq console"]