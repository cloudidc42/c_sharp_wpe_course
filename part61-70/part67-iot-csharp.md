# Part 67: IoT & Hardware with C#
## ขั้นตอนที่ 661-670: C# สำหรับ IoT และ Hardware

---

## 🎯 เป้าหมายของ Part นี้
- .NET IoT Libraries
- GPIO, I2C, SPI, UART
- Raspberry Pi programming
- Sensor reading (Temperature, Humidity, Distance)
- Actuators (LED, Motor, Relay)
- MQTT for IoT messaging
- Real-time sensor dashboard

---

## ขั้นตอนที่ 661: .NET IoT Setup

```bash
# Install for Raspberry Pi / IoT devices
dotnet add package System.Device.Gpio
dotnet add package Iot.Device.Bindings
dotnet add package MQTTnet

# Run on Raspberry Pi:
dotnet publish -r linux-arm64 --self-contained true -p:PublishSingleFile=true
scp app pi@192.168.1.100:/home/pi/
```

```csharp
// Check if running on IoT device
using System.Device.Gpio;
using System.Runtime.InteropServices;

if (!RuntimeInformation.IsOSPlatform(OSPlatform.Linux))
{
    Console.WriteLine("Simulated mode - not running on Linux/Raspberry Pi");
}
```

---

## ขั้นตอนที่ 662: GPIO - LED Control

```csharp
using System.Device.Gpio;

// GPIO pins on Raspberry Pi (BCM numbering)
public class LedController : IDisposable
{
    private readonly GpioController _gpio;
    private readonly int _pin;
    
    public LedController(int pin)
    {
        _gpio = new GpioController();
        _pin = pin;
        _gpio.OpenPin(pin, PinMode.Output);
    }
    
    public void On() => _gpio.Write(_pin, PinValue.High);
    public void Off() => _gpio.Write(_pin, PinValue.Low);
    
    public async Task BlinkAsync(int times = 5, int intervalMs = 500)
    {
        for (int i = 0; i < times; i++)
        {
            On();
            await Task.Delay(intervalMs);
            Off();
            await Task.Delay(intervalMs);
        }
    }
    
    // PWM for brightness control
    public async Task FadeInAsync(int durationMs = 1000)
    {
        for (int brightness = 0; brightness <= 100; brightness += 2)
        {
            // Software PWM simulation
            var onTime = brightness * durationMs / 100 / 50;
            var offTime = (100 - brightness) * durationMs / 100 / 50;
            On();
            await Task.Delay(Math.Max(1, onTime));
            Off();
            await Task.Delay(Math.Max(1, offTime));
        }
    }
    
    public void Dispose()
    {
        Off();
        _gpio.ClosePin(_pin);
        _gpio.Dispose();
    }
}

// Traffic Light simulation
public class TrafficLight : IDisposable
{
    private readonly LedController _red;
    private readonly LedController _yellow;
    private readonly LedController _green;
    
    public TrafficLight(int redPin = 17, int yellowPin = 27, int greenPin = 22)
    {
        _red = new LedController(redPin);
        _yellow = new LedController(yellowPin);
        _green = new LedController(greenPin);
    }
    
    public async Task RunCycleAsync(CancellationToken ct = default)
    {
        while (!ct.IsCancellationRequested)
        {
            // Red (6 seconds)
            SetState(red: true, yellow: false, green: false);
            await Task.Delay(6000, ct);
            
            // Green (4 seconds)
            SetState(red: false, yellow: false, green: true);
            await Task.Delay(4000, ct);
            
            // Yellow (2 seconds)
            SetState(red: false, yellow: true, green: false);
            await Task.Delay(2000, ct);
        }
    }
    
    private void SetState(bool red, bool yellow, bool green)
    {
        if (red) _red.On(); else _red.Off();
        if (yellow) _yellow.On(); else _yellow.Off();
        if (green) _green.On(); else _green.Off();
    }
    
    public void Dispose()
    {
        _red.Dispose(); _yellow.Dispose(); _green.Dispose();
    }
}
```

---

## ขั้นตอนที่ 663: I2C Sensors - DHT22 Temperature & Humidity

```csharp
using Iot.Device.DHTxx;

// DHT22: Temperature and Humidity sensor (uses 1-Wire protocol)
public class TemperatureSensor : IDisposable
{
    private readonly Dht22 _sensor;
    
    public TemperatureSensor(int dataPin = 4)
    {
        _sensor = new Dht22(dataPin);
    }
    
    public SensorReading? Read()
    {
        if (_sensor.TryReadTemperature(out var temp) && 
            _sensor.TryReadHumidity(out var humidity))
        {
            return new SensorReading
            {
                Temperature = temp.DegreesCelsius,
                Humidity = humidity.Percent,
                ReadAt = DateTime.UtcNow
            };
        }
        return null;
    }
    
    public async IAsyncEnumerable<SensorReading> StreamAsync(
        TimeSpan interval,
        [EnumeratorCancellation] CancellationToken ct = default)
    {
        while (!ct.IsCancellationRequested)
        {
            var reading = Read();
            if (reading != null)
                yield return reading;
            
            await Task.Delay(interval, ct);
        }
    }
    
    public void Dispose() => _sensor.Dispose();
}

public record SensorReading(double Temperature, double Humidity, DateTime ReadAt);

// BMP280: Pressure and Altitude sensor (I2C)
public class PressureSensor : IDisposable
{
    private readonly Bmp280 _sensor;
    
    public PressureSensor(int busId = 1, int deviceAddress = 0x76)
    {
        var i2cSettings = new I2cConnectionSettings(busId, deviceAddress);
        var i2cDevice = I2cDevice.Create(i2cSettings);
        _sensor = new Bmp280(i2cDevice);
        _sensor.StandbyTime = StandbyTime.Ms250;
        _sensor.TemperatureSampling = Sampling.LowPower;
        _sensor.PressureSampling = Sampling.UltraHighResolution;
    }
    
    public async Task<(double Temp, double Pressure, double? Altitude)> ReadAsync()
    {
        var result = await _sensor.ReadAsync();
        return (
            result.Temperature!.Value.DegreesCelsius,
            result.Pressure!.Value.Hectopascals,
            result.Altitude?.Meters
        );
    }
    
    public void Dispose() => _sensor.Dispose();
}
```

---

## ขั้นตอนที่ 664: MQTT - IoT Messaging

```csharp
// MQTT: lightweight messaging protocol for IoT
// Install: dotnet add package MQTTnet

using MQTTnet;
using MQTTnet.Client;

// MQTT Publisher (sensor device)
public class SensorPublisher : IAsyncDisposable
{
    private readonly IMqttClient _client;
    private readonly string _brokerHost;
    
    public SensorPublisher(string brokerHost = "broker.hivemq.com")
    {
        _brokerHost = brokerHost;
        var factory = new MqttFactory();
        _client = factory.CreateMqttClient();
    }
    
    public async Task ConnectAsync(string clientId)
    {
        var options = new MqttClientOptionsBuilder()
            .WithClientId(clientId)
            .WithTcpServer(_brokerHost, 1883)
            .WithCleanSession()
            .WithKeepAlivePeriod(TimeSpan.FromSeconds(60))
            .Build();
        
        await _client.ConnectAsync(options);
        Console.WriteLine($"Connected to MQTT broker: {_brokerHost}");
    }
    
    public async Task PublishSensorDataAsync(string deviceId, SensorReading reading)
    {
        var payload = JsonSerializer.Serialize(new
        {
            deviceId,
            temperature = reading.Temperature,
            humidity = reading.Humidity,
            timestamp = reading.ReadAt.ToString("o")
        });
        
        var message = new MqttApplicationMessageBuilder()
            .WithTopic($"sensors/{deviceId}/temperature")
            .WithPayload(payload)
            .WithQualityOfServiceLevel(MqttQualityOfServiceLevel.AtLeastOnce)
            .WithRetainFlag()
            .Build();
        
        await _client.PublishAsync(message);
    }
    
    public async Task StartStreamingAsync(string deviceId, TemperatureSensor sensor, CancellationToken ct)
    {
        await foreach (var reading in sensor.StreamAsync(TimeSpan.FromSeconds(5), ct))
        {
            await PublishSensorDataAsync(deviceId, reading);
            Console.WriteLine($"Published: {reading.Temperature:F1}°C / {reading.Humidity:F1}%");
        }
    }
    
    public async ValueTask DisposeAsync()
    {
        await _client.DisconnectAsync();
        _client.Dispose();
    }
}

// MQTT Subscriber (dashboard server)
public class SensorSubscriber : IAsyncDisposable
{
    private readonly IMqttClient _client;
    public event EventHandler<SensorDataReceivedArgs>? DataReceived;
    
    public SensorSubscriber(string brokerHost = "broker.hivemq.com")
    {
        var factory = new MqttFactory();
        _client = factory.CreateMqttClient();
        
        _client.ApplicationMessageReceivedAsync += OnMessageReceived;
    }
    
    public async Task ConnectAndSubscribeAsync(string pattern = "sensors/+/temperature")
    {
        var options = new MqttClientOptionsBuilder()
            .WithClientId($"dashboard-{Guid.NewGuid():N}")
            .WithTcpServer("broker.hivemq.com", 1883)
            .Build();
        
        await _client.ConnectAsync(options);
        
        var topicFilter = new MqttTopicFilterBuilder()
            .WithTopic(pattern)
            .WithQualityOfServiceLevel(MqttQualityOfServiceLevel.AtLeastOnce)
            .Build();
        
        await _client.SubscribeAsync(topicFilter);
        Console.WriteLine($"Subscribed to {pattern}");
    }
    
    private Task OnMessageReceived(MqttApplicationMessageReceivedEventArgs e)
    {
        var payload = Encoding.UTF8.GetString(e.ApplicationMessage.PayloadSegment);
        var data = JsonSerializer.Deserialize<SensorPayload>(payload);
        
        if (data != null)
            DataReceived?.Invoke(this, new SensorDataReceivedArgs(e.ApplicationMessage.Topic, data));
        
        return Task.CompletedTask;
    }
    
    public async ValueTask DisposeAsync()
    {
        await _client.DisconnectAsync();
        _client.Dispose();
    }
}
```

---

## ขั้นตอนที่ 665: Serial Port / UART

```csharp
using System.IO.Ports;

// UART: communicate with Arduino, ESP32, etc.
public class SerialCommunicator : IDisposable
{
    private readonly SerialPort _port;
    private readonly Channel<string> _received = Channel.CreateUnbounded<string>();
    
    public SerialCommunicator(string portName = "/dev/ttyUSB0", int baudRate = 9600)
    {
        _port = new SerialPort(portName, baudRate, Parity.None, 8, StopBits.One);
        _port.NewLine = "\n";
        _port.DataReceived += OnDataReceived;
    }
    
    public void Open()
    {
        _port.Open();
        Console.WriteLine($"Opened {_port.PortName} at {_port.BaudRate} baud");
    }
    
    private void OnDataReceived(object sender, SerialDataReceivedEventArgs e)
    {
        try
        {
            var line = _port.ReadLine().Trim();
            _received.Writer.TryWrite(line);
        }
        catch (Exception ex)
        {
            Console.WriteLine($"Read error: {ex.Message}");
        }
    }
    
    public async IAsyncEnumerable<string> ReadLinesAsync([EnumeratorCancellation] CancellationToken ct = default)
    {
        await foreach (var line in _received.Reader.ReadAllAsync(ct))
            yield return line;
    }
    
    public async Task SendCommandAsync(string command)
    {
        await Task.Run(() => _port.WriteLine(command));
    }
    
    public void Dispose()
    {
        _port.DataReceived -= OnDataReceived;
        _port.Close();
        _port.Dispose();
    }
}

// Parse sensor data from Arduino
// Arduino sends: "T:23.5,H:65.2,P:1013.2"
public static SensorData ParseArduinoData(string raw)
{
    var parts = raw.Split(',');
    var data = new SensorData();
    foreach (var part in parts)
    {
        var kv = part.Split(':');
        if (kv.Length == 2 && double.TryParse(kv[1], out var value))
        {
            switch (kv[0])
            {
                case "T": data.Temperature = value; break;
                case "H": data.Humidity = value; break;
                case "P": data.Pressure = value; break;
            }
        }
    }
    return data;
}
```

---

## ขั้นตอนที่ 666-670: IoT Dashboard

```csharp
// IoT Sensor Dashboard - WPF app showing real-time sensor data
public class SensorDashboardViewModel : ViewModelBase, IAsyncDisposable
{
    private readonly SensorSubscriber _subscriber;
    private readonly CancellationTokenSource _cts = new();
    
    public SensorDashboardViewModel()
    {
        _subscriber = new SensorSubscriber();
        _subscriber.DataReceived += OnDataReceived;
        Devices = new ObservableCollection<DeviceViewModel>();
        ConnectCommand = new AsyncRelayCommand(ConnectAsync);
    }
    
    public ObservableCollection<DeviceViewModel> Devices { get; }
    public IAsyncRelayCommand ConnectCommand { get; }
    
    private bool _isConnected;
    public bool IsConnected { get => _isConnected; set => SetProperty(ref _isConnected, value); }
    
    private async Task ConnectAsync()
    {
        await _subscriber.ConnectAndSubscribeAsync("sensors/#");
        IsConnected = true;
    }
    
    private void OnDataReceived(object? sender, SensorDataReceivedArgs e)
    {
        App.Current.Dispatcher.Invoke(() =>
        {
            var device = Devices.FirstOrDefault(d => d.DeviceId == e.Data.DeviceId);
            if (device == null)
            {
                device = new DeviceViewModel(e.Data.DeviceId);
                Devices.Add(device);
            }
            device.Update(e.Data);
        });
    }
    
    public async ValueTask DisposeAsync()
    {
        _cts.Cancel();
        await _subscriber.DisposeAsync();
    }
}

public class DeviceViewModel : ViewModelBase
{
    public string DeviceId { get; }
    
    private double _temperature;
    public double Temperature { get => _temperature; set => SetProperty(ref _temperature, value); }
    
    private double _humidity;
    public double Humidity { get => _humidity; set => SetProperty(ref _humidity, value); }
    
    private DateTime _lastSeen;
    public DateTime LastSeen { get => _lastSeen; set => SetProperty(ref _lastSeen, value); }
    
    public ObservableCollection<TemperaturePoint> History { get; } = new();
    
    public DeviceViewModel(string deviceId) => DeviceId = deviceId;
    
    public void Update(SensorPayload data)
    {
        Temperature = data.Temperature;
        Humidity = data.Humidity;
        LastSeen = DateTime.Now;
        History.Add(new TemperaturePoint(DateTime.Now, data.Temperature));
        if (History.Count > 60) History.RemoveAt(0); // keep last 60 readings
    }
}

public record TemperaturePoint(DateTime Time, double Value);
```

---

## 📝 สรุป Part 67

| Technology | ใช้กับ |
|-----------|-------|
| System.Device.Gpio | GPIO pins (LED, Button, Relay) |
| Iot.Device.Bindings | Sensors (DHT22, BMP280, etc.) |
| SerialPort | UART/RS232 (Arduino) |
| MQTTnet | IoT messaging |
| MQTT broker | HiveMQ, Mosquitto, AWS IoT |

แพลตฟอร์มที่รองรับ .NET IoT:
- Raspberry Pi (Linux ARM)
- Orange Pi, Rock Pi
- Arduino + ESP32 (via Serial)
- Windows IoT (Raspberry Pi 4)

---

**ก่อนหน้า → [Part 66: .NET MAUI](part66-maui.md)**  
**ต่อไป → [Part 68: Roslyn & Code Generation](part68-roslyn.md)**
