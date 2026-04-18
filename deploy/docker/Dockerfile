ARG FLINK_VERSION
ARG DINKY_VERSION

# ── Stage 1: 解压 Dinky 产物 ──────────────────────────────────────────
FROM flink:${FLINK_VERSION}-scala_2.12-java11 AS builder
ADD build/dinky-release*.tar.gz /opt
RUN mv /opt/dinky-release* /opt/dinky && \
    mkdir -p /opt/dinky/config/static
ADD build/dist/ /opt/dinky/config/static

# ── Stage 2: 最终镜像 ─────────────────────────────────────────────────
FROM flink:${FLINK_VERSION}-scala_2.12-java11

ARG FLINK_VERSION

# ── 文件系统插件 → plugins/ ───────────────────────────────────────────
# ── 文件系统插件 → plugins/ ───────────────────────────────────────────
RUN mkdir -p $FLINK_HOME/plugins/s3-fs-hadoop \
    $FLINK_HOME/plugins/oss-fs-hadoop && \
    cp $FLINK_HOME/opt/flink-s3-fs-hadoop-*.jar \
    $FLINK_HOME/plugins/s3-fs-hadoop/ && \
    cp $FLINK_HOME/opt/flink-oss-fs-hadoop-*.jar \
    $FLINK_HOME/plugins/oss-fs-hadoop/

# ── 可选组件 → lib/ ───────────────────────────────────────────────────
# 替换 table-planner-loader 为 Scala 版 table-planner
RUN rm -f $FLINK_HOME/lib/flink-table-planner-loader-*.jar && \
    cp $FLINK_HOME/opt/flink-cep-scala_*.jar \
    $FLINK_HOME/opt/flink-state-processor-api-*.jar \
    $FLINK_HOME/opt/flink-sql-gateway-*.jar \
    $FLINK_HOME/opt/flink-python-*.jar \
    $FLINK_HOME/opt/flink-table-planner_*.jar \
    $FLINK_HOME/lib/

# ── PyFlink Python 环境 ───────────────────────────────────────────────
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    python3 \
    python3-pip \
    gcc \
    python3-dev && \
    pip3 install --no-cache-dir \
    # PyFlink 核心
    apache-flink==${FLINK_VERSION} \
    # Pandas UDF 必须（ETL/聚合计算常用）
    pandas \
    pyarrow \
    # Kafka 消息序列化
    confluent-kafka \
    fastavro \
    protobuf \
    # Parquet/ORC/Avro 格式
    pyarrow \
    fastparquet \
    # JDBC/CDC 数据处理
    sqlalchemy \
    pymysql \
    psycopg2-binary \
    # 时间处理
    python-dateutil && \
    # 编译完成后移除编译工具，减小镜像体积
    apt-get purge -y gcc python3-dev && \
    apt-get autoremove -y && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# ── 验证 ──────────────────────────────────────────────────────────────
RUN echo "=== plugins ===" && \
    find $FLINK_HOME/plugins -name "*.jar" && \
    echo "=== lib (新增) ===" && \
    ls $FLINK_HOME/lib/flink-cep* \
    $FLINK_HOME/lib/flink-state-processor* \
    $FLINK_HOME/lib/flink-sql-gateway* \
    $FLINK_HOME/lib/flink-python* \
    $FLINK_HOME/lib/flink-table-planner_*

# ── Dinky ─────────────────────────────────────────────────────────────
COPY --from=builder /opt/dinky /opt/dinky

ENV FLINK_VERSION=${FLINK_VERSION}
ENV DINKY_HOME=/opt/dinky/
ENV H2_DB=./tmp/db/h2

USER root

RUN ln -s $FLINK_HOME/lib/* /opt/dinky/extends/flink${FLINK_VERSION}/ && \
    mkdir -p /opt/dinky/customJar /dinky && \
    chmod -R 777 /opt/dinky/ /dinky/ && \
    sed -i 's/-Xms512M -Xmx2048M -XX:PermSize=512M/-XX:+UseContainerSupport -XX:InitialRAMPercentage=70.0 -XX:MaxRAMPercentage=70.0/g' \
    /opt/dinky/bin/auto.sh

WORKDIR /opt/dinky/

EXPOSE 8888

CMD ./bin/auto.sh startOnPending