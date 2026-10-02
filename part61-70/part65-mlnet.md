# Part 65: ML.NET - Machine Learning ใน C#
## ขั้นตอนที่ 641-650: เพิ่ม AI/ML ในแอปพลิเคชัน

---

## 🎯 เป้าหมายของ Part นี้
- ML.NET Framework overview
- Binary Classification (spam detection)
- Multi-class Classification (product categorization)
- Regression (price prediction)
- Recommendation System
- Anomaly Detection
- Integrate ML model ใน WPF app

---

## ขั้นตอนที่ 641: ML.NET Overview

```
ML.NET Pipeline:
Data → Load → Transform → Train → Evaluate → Save → Predict

ติดตั้ง packages:
dotnet add package Microsoft.ML
dotnet add package Microsoft.ML.FastTree
dotnet add package Microsoft.ML.LightGbm
dotnet add package Microsoft.ML.TimeSeries
```

---

## ขั้นตอนที่ 642: Binary Classification - Spam Detection

```csharp
// Spam detection model
// Install: Microsoft.ML

using Microsoft.ML;
using Microsoft.ML.Data;

// Input class (training data)
public class EmailData
{
    [LoadColumn(0)] public string? Text { get; set; }
    [LoadColumn(1), ColumnName("Label")] public bool IsSpam { get; set; }
}

// Prediction output
public class SpamPrediction
{
    [ColumnName("PredictedLabel")] public bool IsSpam { get; set; }
    public float Probability { get; set; }
    public float Score { get; set; }
}

public class SpamDetector
{
    private readonly MLContext _ml = new MLContext(seed: 42);
    private ITransformer? _model;
    private PredictionEngine<EmailData, SpamPrediction>? _engine;
    
    public void Train(string dataPath)
    {
        // Load data
        var data = _ml.Data.LoadFromTextFile<EmailData>(dataPath,
            separatorChar: '\t', hasHeader: true);
        
        var (train, test) = _ml.Data.TrainTestSplit(data, testFraction: 0.2);
        
        // Build pipeline
        var pipeline = _ml.Transforms.Text
            .FeaturizeText("Features", nameof(EmailData.Text))
            .Append(_ml.BinaryClassification.Trainers.FastTree(
                labelColumnName: "Label",
                featureColumnName: "Features",
                numberOfLeaves: 50,
                learningRate: 0.1f));
        
        // Train
        Console.WriteLine("Training spam detector...");
        _model = pipeline.Fit(train);
        
        // Evaluate
        var predictions = _model.Transform(test);
        var metrics = _ml.BinaryClassification.Evaluate(predictions, labelColumnName: "Label");
        
        Console.WriteLine($"Accuracy: {metrics.Accuracy:P2}");
        Console.WriteLine($"F1 Score: {metrics.F1Score:P2}");
        Console.WriteLine($"AUC: {metrics.AreaUnderRocCurve:P2}");
        
        // Create prediction engine (not thread-safe, use pool)
        _engine = _ml.Model.CreatePredictionEngine<EmailData, SpamPrediction>(_model);
    }
    
    public void SaveModel(string path)
        => _ml.Model.Save(_model, null, path);
    
    public void LoadModel(string path)
    {
        _model = _ml.Model.Load(path, out _);
        _engine = _ml.Model.CreatePredictionEngine<EmailData, SpamPrediction>(_model);
    }
    
    public SpamPrediction Predict(string emailText)
    {
        var input = new EmailData { Text = emailText };
        return _engine!.Predict(input);
    }
}

// Usage
var detector = new SpamDetector();
detector.Train("data/emails.tsv");
detector.SaveModel("models/spam_detector.zip");

var samples = new[]
{
    "Meeting tomorrow at 3pm in conference room A",
    "CONGRATULATIONS! You won $1000000! Click here to claim your prize NOW!",
    "Please review the quarterly report attached",
    "FREE BITCOIN! Limited time offer! Act now!"
};

Console.WriteLine("\n=== Spam Detection Results ===");
foreach (var text in samples)
{
    var result = detector.Predict(text);
    var label = result.IsSpam ? "🚫 SPAM" : "✅ Ham";
    Console.WriteLine($"{label} ({result.Probability:P0}): {text[..Math.Min(50, text.Length)]}...");
}
```

---

## ขั้นตอนที่ 643: Regression - Price Prediction

```csharp
// Predict house/product price
public class HouseData
{
    [LoadColumn(0)] public float Size { get; set; }           // ตรม
    [LoadColumn(1)] public float Bedrooms { get; set; }       // ห้องนอน
    [LoadColumn(2)] public float Bathrooms { get; set; }      // ห้องน้ำ
    [LoadColumn(3)] public float DistanceToCenter { get; set; } // กม.
    [LoadColumn(4)] public float Age { get; set; }             // อายุบ้าน
    [LoadColumn(5), ColumnName("Label")] public float Price { get; set; }
}

public class HousePricePrediction
{
    [ColumnName("Score")] public float Price { get; set; }
}

public class HousePricePredictor
{
    private readonly MLContext _ml = new MLContext(seed: 42);
    private ITransformer? _model;
    
    public void Train(IEnumerable<HouseData> data)
    {
        var dataView = _ml.Data.LoadFromEnumerable(data);
        var split = _ml.Data.TrainTestSplit(dataView, testFraction: 0.2);
        
        var features = new[] 
        { 
            nameof(HouseData.Size), nameof(HouseData.Bedrooms),
            nameof(HouseData.Bathrooms), nameof(HouseData.DistanceToCenter), 
            nameof(HouseData.Age)
        };
        
        var pipeline = _ml.Transforms.Concatenate("Features", features)
            .Append(_ml.Transforms.NormalizeMinMax("Features"))
            .Append(_ml.Regression.Trainers.FastForest(
                labelColumnName: "Label",
                featureColumnName: "Features",
                numberOfLeaves: 100,
                numberOfTrees: 200));
        
        _model = pipeline.Fit(split.TrainSet);
        
        var predictions = _model.Transform(split.TestSet);
        var metrics = _ml.Regression.Evaluate(predictions, labelColumnName: "Label");
        
        Console.WriteLine($"R²: {metrics.RSquared:F4} (1.0 = perfect)");
        Console.WriteLine($"RMSE: ฿{metrics.RootMeanSquaredError:N0}");
        Console.WriteLine($"MAE: ฿{metrics.MeanAbsoluteError:N0}");
    }
    
    public float PredictPrice(HouseData house)
    {
        var engine = _ml.Model.CreatePredictionEngine<HouseData, HousePricePrediction>(_model!);
        return engine.Predict(house).Price;
    }
}

// Usage
var data = GenerateHouseData();
var predictor = new HousePricePredictor();
predictor.Train(data);

var predicted = predictor.PredictPrice(new HouseData
{
    Size = 120, Bedrooms = 3, Bathrooms = 2,
    DistanceToCenter = 15, Age = 5
});
Console.WriteLine($"ราคาบ้านที่คาดการณ์: ฿{predicted:N0}");
```

---

## ขั้นตอนที่ 644: Recommendation System

```csharp
// Product recommendation using Matrix Factorization
public class ProductRating
{
    public float UserId { get; set; }
    public float ProductId { get; set; }
    public float Rating { get; set; } // 1-5
}

public class RatingPrediction
{
    public float Score { get; set; }
}

public class RecommendationEngine
{
    private readonly MLContext _ml = new MLContext(seed: 42);
    private ITransformer? _model;
    
    public void Train(IEnumerable<ProductRating> ratings)
    {
        var data = _ml.Data.LoadFromEnumerable(ratings);
        
        var pipeline = _ml.Transforms.Conversion.MapValueToKey("UserIdEncoded", "UserId")
            .Append(_ml.Transforms.Conversion.MapValueToKey("ProductIdEncoded", "ProductId"))
            .Append(_ml.Recommendation().Trainers.MatrixFactorization(
                labelColumnName: "Rating",
                matrixColumnIndexColumnName: "UserIdEncoded",
                matrixRowIndexColumnName: "ProductIdEncoded",
                numberOfIterations: 20,
                approximationRank: 100));
        
        _model = pipeline.Fit(data);
    }
    
    public List<(int ProductId, float PredictedRating)> GetRecommendations(
        int userId, IEnumerable<int> productIds, int topN = 10)
    {
        var engine = _ml.Model.CreatePredictionEngine<ProductRating, RatingPrediction>(_model!);
        
        return productIds
            .Select(pid => (ProductId: pid, Rating: engine.Predict(new ProductRating
            {
                UserId = userId,
                ProductId = pid
            }).Score))
            .OrderByDescending(x => x.Rating)
            .Take(topN)
            .ToList();
    }
}
```

---

## ขั้นตอนที่ 645: Anomaly Detection

```csharp
// Detect unusual sales patterns
public class SalesDataPoint
{
    public string? Month { get; set; }
    public float Sales { get; set; }
}

public class SalesAnomalyDetector
{
    private readonly MLContext _ml = new MLContext();
    
    public List<(string Month, float Sales, bool IsAnomaly, double Score)> 
        DetectAnomalies(IEnumerable<SalesDataPoint> salesData)
    {
        var data = _ml.Data.LoadFromEnumerable(salesData);
        
        // Spike detection
        var spikeEstimator = _ml.Transforms.DetectIidSpike(
            outputColumnName: "Prediction",
            inputColumnName: "Sales",
            confidence: 95,
            pvalueHistoryLength: salesData.Count() / 4);
        
        var spikeTransform = spikeEstimator.Fit(data);
        var predictions = spikeTransform.Transform(data);
        
        var results = _ml.Data.CreateEnumerable<SalesAnomalyResult>(predictions, reuseRowObject: false);
        
        return salesData.Zip(results, (s, r) => (
            Month: s.Month!,
            Sales: s.Sales,
            IsAnomaly: r.Prediction[0] == 1,
            Score: r.Prediction[1]
        )).ToList();
    }
    
    private class SalesAnomalyResult
    {
        [VectorType(3)] public double[]? Prediction { get; set; }
    }
}

// Usage
var detector = new SalesAnomalyDetector();
var monthlySales = Enumerable.Range(1, 24).Select(i => new SalesDataPoint
{
    Month = $"2024-{i % 12 + 1:D2}",
    Sales = 50000 + (float)(new Random(i).NextDouble() * 10000) + (i == 13 ? 150000 : 0) // spike at month 13
}).ToList();

var results = detector.DetectAnomalies(monthlySales);
Console.WriteLine("=== Anomaly Detection ===");
foreach (var r in results)
{
    var flag = r.IsAnomaly ? "🚨 ANOMALY" : "✅ Normal";
    Console.WriteLine($"{r.Month}: ฿{r.Sales:N0} - {flag} (score: {r.Score:F2})");
}
```

---

## ขั้นตอนที่ 646-650: ML Model ใน WPF App

```csharp
// PricePredictionViewModel.cs - Use ML model in WPF MVVM
public class PricePredictionViewModel : ViewModelBase
{
    private readonly HousePricePredictor _predictor;
    
    public PricePredictionViewModel()
    {
        _predictor = new HousePricePredictor();
        LoadModelAsync().FireAndForget();
        PredictCommand = new AsyncRelayCommand(PredictAsync);
    }
    
    private float _size = 120;
    public float Size { get => _size; set { SetProperty(ref _size, value); } }
    
    private float _bedrooms = 3;
    public float Bedrooms { get => _bedrooms; set { SetProperty(ref _bedrooms, value); } }
    
    private float _bathrooms = 2;
    public float Bathrooms { get => _bathrooms; set { SetProperty(ref _bathrooms, value); } }
    
    private decimal _predictedPrice;
    public decimal PredictedPrice { get => _predictedPrice; set => SetProperty(ref _predictedPrice, value); }
    
    private bool _isModelLoaded;
    public bool IsModelLoaded { get => _isModelLoaded; set => SetProperty(ref _isModelLoaded, value); }
    
    public IAsyncRelayCommand PredictCommand { get; }
    
    private async Task LoadModelAsync()
    {
        await Task.Run(() => {
            var modelPath = "models/house_price.zip";
            if (File.Exists(modelPath))
                _predictor.LoadModel(modelPath);
            else
            {
                var data = LoadTrainingData();
                _predictor.Train(data);
                _predictor.SaveModel(modelPath);
            }
        });
        IsModelLoaded = true;
    }
    
    private async Task PredictAsync()
    {
        var price = await Task.Run(() => _predictor.PredictPrice(new HouseData
        {
            Size = Size,
            Bedrooms = Bedrooms,
            Bathrooms = Bathrooms,
            DistanceToCenter = 10,
            Age = 5
        }));
        PredictedPrice = (decimal)price;
    }
    
    private IEnumerable<HouseData> LoadTrainingData()
    {
        // Generate synthetic training data
        var rng = new Random(42);
        return Enumerable.Range(0, 5000).Select(_ =>
        {
            var size = (float)rng.Next(50, 300);
            var beds = (float)rng.Next(1, 6);
            var baths = (float)rng.Next(1, 4);
            var dist = (float)(rng.NextDouble() * 50);
            var age = (float)rng.Next(0, 40);
            // Price formula with noise
            var price = (size * 12000 + beds * 100000 + baths * 50000 
                        - dist * 5000 - age * 2000) * (0.9f + (float)rng.NextDouble() * 0.2f);
            return new HouseData { Size = size, Bedrooms = beds, Bathrooms = baths,
                DistanceToCenter = dist, Age = age, Price = Math.Max(price, 500000) };
        });
    }
}
```

```xml
<!-- PricePredictionView.xaml -->
<Window xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="ML.NET Price Predictor" Width="400" Height="450">
    <Grid Margin="20">
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="*"/>
            <RowDefinition Height="Auto"/>
        </Grid.RowDefinitions>
        
        <TextBlock Grid.Row="0" Text="🏠 ทำนายราคาบ้าน" FontSize="20" FontWeight="Bold" Margin="0,0,0,20"/>
        
        <!-- Size -->
        <StackPanel Grid.Row="1" Margin="0,0,0,10">
            <TextBlock Text="ขนาดบ้าน (ตรม.)"/>
            <Slider Value="{Binding Size}" Minimum="30" Maximum="500" TickFrequency="10"/>
            <TextBlock Text="{Binding Size, StringFormat='{}{0:N0} ตรม.'}"/>
        </StackPanel>
        
        <!-- Bedrooms -->
        <StackPanel Grid.Row="2" Margin="0,0,0,10">
            <TextBlock Text="ห้องนอน"/>
            <Slider Value="{Binding Bedrooms}" Minimum="1" Maximum="8" TickFrequency="1"/>
            <TextBlock Text="{Binding Bedrooms, StringFormat='{}{0:N0} ห้อง'}"/>
        </StackPanel>
        
        <!-- Bathrooms -->
        <StackPanel Grid.Row="3" Margin="0,0,0,10">
            <TextBlock Text="ห้องน้ำ"/>
            <Slider Value="{Binding Bathrooms}" Minimum="1" Maximum="5" TickFrequency="1"/>
            <TextBlock Text="{Binding Bathrooms, StringFormat='{}{0:N0} ห้อง'}"/>
        </StackPanel>
        
        <!-- Predict button -->
        <Button Grid.Row="4" Content="ทำนายราคา" 
                Command="{Binding PredictCommand}"
                IsEnabled="{Binding IsModelLoaded}"
                Padding="10" Margin="0,10,0,0"/>
        
        <!-- Result -->
        <StackPanel Grid.Row="6" Margin="0,10,0,0">
            <TextBlock Text="ราคาที่คาดการณ์:" FontSize="14"/>
            <TextBlock FontSize="28" FontWeight="Bold" Foreground="DarkGreen"
                       Text="{Binding PredictedPrice, StringFormat='฿{0:N0}'}"/>
        </StackPanel>
    </Grid>
</Window>
```

---

## 📝 สรุป Part 65

| ML Task | Trainer | ใช้กับ |
|---------|---------|-------|
| Binary Classification | FastTree | Spam, Fraud detection |
| Multi-class Classification | LightGbm | Category labeling |
| Regression | FastForest | Price prediction |
| Recommendation | MatrixFactorization | Product suggestions |
| Anomaly Detection | IidSpike | Fraud, Outlier detection |

---

**ก่อนหน้า → [Part 64: Microservices](part64-microservices.md)**  
**ต่อไป → [Part 66: .NET MAUI](part66-maui.md)**
