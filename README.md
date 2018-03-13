# CFS-SensorTag  

Connected Field Service デモ by TI SensorTag のセットアップ手順を説明します。

1. SensorTag から Azure IoT Hub まで接続する  
  * 以下に記載の手順を実行  
    Use the Azure IoT Gateway SDK to send device-to-cloud messages with a physical device (Linux)  
    https://docs.microsoft.com/en-us/azure/iot-hub/iot-hub-gateway-sdk-physical-device  
  * この時点で、センサーデータは IoT Hub に届くが、まだ Azure Stream Analytics ジョブで扱うことができない。データが JSON 形式ではないため。  
2. センサーデータを JSON 形式で Azure IoT Hub に出力する  
  * 以下の Extension をダウンロードして、Raspberry Pi 3 上の SDK を上書き  
    BLE Sample Extension of Azure IoT Gateway SDK  
    https://github.com/ms-iotkithol-jp/AzureIoTGatewaySDKExtention  
  * 以下に記載のサンプル ble_json_gateway を実行  
    Raspberry Pi3(Raspbian) + TI Sensor Tag CC2650 Sample  
    https://github.com/ms-iotkithol-jp/AzureIoTGatewaySDKExtention/blob/master/samples/ble_json_gateway/src/readme.md  
  * この時点で、センサーデータは Azure Stream Analytics ジョブで扱うことができるが、まだクエリの修正が必要。  
3. クエリの編集  
  * 以下はクエリのサンプルです。
    * 「PowerBIkeijidemo01」という出力は、Power BI リアルタイム ストリームデータ 可視化のためのものです。必要ない場合には最後の SELECT 文全体が不要です。  
    * Photon による接続にも、CFS 標準のシミュレーターによる接続にも対応できるクエリになっています。SensorTag 専用のクエリにする場合にはもう少しシンプルになります。  
    ```  
    WITH TelemetryData AS
    (
    SELECT
        CASE
            WHEN Stream.device_id IS NOT null THEN Stream.device_id
            ELSE CASE
                WHEN Stream.IoTHub.ConnectionDeviceId IS NOT null THEN Stream.IoTHub.ConnectionDeviceId
                ELSE Stream.DeviceID
            END
        END as DeviceID,
            'Temperature' AS ReadingType,
        CASE
            WHEN Stream.data IS NOT null THEN Stream.data
            ELSE CASE
                WHEN Stream.ambience IS NOT null THEN Stream.ambience
                ELSE Stream.Temperature
            END
        END as Reading,
            Stream.EventToken AS EventToken,
            Ref.Temperature AS Threshold,
            Ref.TemperatureRuleOutput AS RuleOutput,
            Stream.EventEnqueuedUtcTime AS [Time],
        CASE
            WHEN Stream.gyrox IS NOT null THEN CAST ( Stream.gyrox AS float )
            ELSE 0.0
        END as gyrox,
        CASE
            WHEN Stream.gyroy IS NOT null THEN CAST ( Stream.gyroy AS float )
            ELSE 0.0
        END as gyroy,
        CASE
            WHEN Stream.gyroz IS NOT null THEN CAST ( Stream.gyroz AS float )
            ELSE 0.0
        END as gyroz
    FROM IoTStream Stream
    JOIN DeviceRulesBlob Ref ON Ref.DeviceType = 'Thermostat'
    ),
    MaxInMinute AS
    (
    SELECT
        TopOne() OVER (ORDER BY Reading DESC) AS telemetryEvent
    FROM
        TelemetryData
    GROUP BY
        TumblingWindow(minute, 1), DeviceId
    )

    SELECT telemetryEvent.DeviceId,
    	telemetryEvent.ReadingType,
    	telemetryEvent.Reading,
    	telemetryEvent.EventToken,
    	telemetryEvent.Threshold,
    	telemetryEvent.RuleOutput,
    	telemetryEvent.Time
    INTO PowerBISQL
    FROM MaxInMinute

    SELECT DeviceId,
    	ReadingType,
    	Reading,
        gyrox AS Gyro_X,
        gyroy AS Gyro_Y,
        gyroz AS Gyro_Z,
    	EventToken,
    	Threshold,
    	RuleOutput,
    	Time
    INTO PowerBIkeijidemo01
    FROM TelemetryData
    ```  
