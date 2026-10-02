# Part 90: Capstone - ML & AI Integration
## ขั้นตอนที่ 891-900: AI-Powered ShopThai

---

## 🎯 เป้าหมายของ Part นี้
- ML.NET สำหรับ in-process ML
- Semantic Kernel สำหรับ LLM integration
- Product recommendation engine
- Sentiment analysis ของ reviews
- Smart pricing prediction
- AI customer service chatbot
- Vector search ด้วย pgvector

---

## ขั้นตอนที่ 891: ML.NET Recommendation Engine

```csharp
// dotnet add package Microsoft.ML
// dotnet add package Microsoft.ML.Recommender

public class ProductRecommendationService
{
    private readonly MLContext _ml = new();
    private ITransformer? _model;
    
    // Training data schema
    public class ProductRating
    {
        public float UserId { get; set; }
        public float ProductId { get; set; }
        public float Label { get; set; } // rating or implicit (1 = viewed/purchased)
    }
    
    public class ProductPrediction
    {
        public float Score { get; set; }
    }
    
    public void Train(IEnumerable<ProductRating> ratings)
    {
        var data = _ml.Data.LoadFromEnumerable(ratings);
        
        var pipeline = _ml.Transforms.Conversion
            .MapValueToKey("UserIdEncoded", "UserId")
            .Append(_ml.Transforms.Conversion.MapValueToKey("ProductIdEncoded", "ProductId"))
            .Append(_ml.Recommendation().Trainers.MatrixFactorization(
                labelColumnName: "Label",
                matrixColumnIndexColumnName: "UserIdEncoded",
                matrixRowIndexColumnName: "ProductIdEncoded",
                numberOfIterations: 20,
                approximationRank: 100));
        
        _model = pipeline.Fit(data);
    }
    
    public IEnumerable<(int ProductId, float Score)> Recommend(int userId, IEnumerable<int> candidateProductIds)
    {
        if (_model == null) throw new InvalidOperationException("Model not trained");
        
        var engine = _ml.Model.CreatePredictionEngine<ProductRating, ProductPrediction>(_model);
        
        return candidateProductIds
            .Select(pid => (ProductId: pid, Score: engine.Predict(new ProductRating
            {
                UserId = userId,
                ProductId = pid
            }).Score))
            .OrderByDescending(x => x.Score)
            .Take(10);
    }
    
    public void SaveModel(string path) => _ml.Model.Save(_model, null, path);
    public void LoadModel(string path) => _model = _ml.Model.Load(path, out _);
}
```

---

## ขั้นตอนที่ 892: Sentiment Analysis

```csharp
// ML.NET sentiment analysis for product reviews
public class SentimentAnalysisService
{
    private readonly MLContext _ml = new();
    private ITransformer? _model;
    
    public class ReviewData
    {
        [LoadColumn(0)] public string ReviewText { get; set; } = "";
        [LoadColumn(1), ColumnName("Label")] public bool IsPositive { get; set; }
    }
    
    public class ReviewPrediction
    {
        [ColumnName("PredictedLabel")] public bool IsPositive { get; set; }
        public float Probability { get; set; }
        public float Score { get; set; }
    }
    
    public void Train(string dataPath)
    {
        var data = _ml.Data.LoadFromTextFile<ReviewData>(dataPath, separatorChar: '\t');
        var split = _ml.Data.TrainTestSplit(data, testFraction: 0.2);
        
        var pipeline = _ml.Transforms.Text
            .FeaturizeText("Features", "ReviewText")
            .Append(_ml.BinaryClassification.Trainers
                .SdcaLogisticRegression(labelColumnName: "Label", featureColumnName: "Features"));
        
        _model = pipeline.Fit(split.TrainSet);
        
        // Evaluate
        var predictions = _model.Transform(split.TestSet);
        var metrics = _ml.BinaryClassification.Evaluate(predictions);
        _logger.LogInformation("Model AUC: {AUC:P2}, Accuracy: {Acc:P2}", 
            metrics.AreaUnderRocCurve, metrics.Accuracy);
    }
    
    public (bool IsPositive, float Confidence) Analyze(string reviewText)
    {
        var engine = _ml.Model.CreatePredictionEngine<ReviewData, ReviewPrediction>(_model!);
        var prediction = engine.Predict(new ReviewData { ReviewText = reviewText });
        return (prediction.IsPositive, prediction.Probability);
    }
}

// Auto-moderate reviews
public class ReviewModerationService
{
    public async Task<ReviewModerationResult> ModerateAsync(string reviewText)
    {
        var (isPositive, confidence) = _sentiment.Analyze(reviewText);
        
        // Flag for manual review if low confidence
        if (confidence < 0.7f)
            return ReviewModerationResult.PendingReview;
        
        // Extremely negative with spam keywords → reject
        if (!isPositive && confidence > 0.95f && ContainsSpam(reviewText))
            return ReviewModerationResult.Rejected;
        
        return ReviewModerationResult.Approved;
    }
}
```

---

## ขั้นตอนที่ 893: Semantic Kernel & LLM Integration

```csharp
// dotnet add package Microsoft.SemanticKernel

public class AiProductDescriptionService
{
    private readonly Kernel _kernel;
    
    public AiProductDescriptionService(IConfiguration config)
    {
        _kernel = Kernel.CreateBuilder()
            .AddOpenAIChatCompletion(
                modelId: "gpt-4o-mini",
                apiKey: config["OpenAI:ApiKey"]!)
            .Build();
    }
    
    public async Task<string> GenerateDescriptionAsync(Product product)
    {
        var prompt = $"""
            Generate a compelling Thai product description for an e-commerce site.
            
            Product: {product.Name}
            Category: {product.Category}
            Price: ฿{product.Price:N0}
            Key features: {string.Join(", ", product.Features)}
            
            Requirements:
            - 2-3 paragraphs in Thai
            - Highlight key benefits
            - Include call-to-action
            - SEO-friendly
            """;
        
        var result = await _kernel.InvokePromptAsync(prompt);
        return result.GetValue<string>()!;
    }
    
    // Semantic Function with template
    public async Task<string> SummarizeReviewsAsync(IEnumerable<string> reviews)
    {
        var summarizeFunc = _kernel.CreateFunctionFromPrompt("""
            Summarize the following customer reviews in Thai in 2 sentences,
            highlighting the most common positive and negative points:
            
            {{$reviews}}
            """);
        
        var result = await _kernel.InvokeAsync(summarizeFunc, new KernelArguments
        {
            ["reviews"] = string.Join("\n---\n", reviews.Take(20))
        });
        
        return result.GetValue<string>()!;
    }
}
```

---

## ขั้นตอนที่ 894: Vector Search ด้วย pgvector

```csharp
// dotnet add package Pgvector.EntityFrameworkCore
// PostgreSQL: CREATE EXTENSION vector;

public class ProductEmbeddingService
{
    private readonly Kernel _kernel;
    private readonly AppDbContext _db;
    
    // Generate embedding vector for product
    public async Task<float[]> GetEmbeddingAsync(string text)
    {
        var embeddings = _kernel.Services.GetRequiredService<ITextEmbeddingGenerationService>();
        var result = await embeddings.GenerateEmbeddingAsync(text);
        return result.ToArray();
    }
    
    // Index product with semantic embedding
    public async Task IndexProductAsync(Product product)
    {
        var text = $"{product.Name} {product.Description} {string.Join(" ", product.Tags)}";
        var embedding = await GetEmbeddingAsync(text);
        
        product.SearchVector = new Vector(embedding);
        await _db.SaveChangesAsync();
    }
    
    // Semantic search: "รองเท้าวิ่งใส่สบาย" → finds relevant products
    public async Task<IEnumerable<Product>> SemanticSearchAsync(string query, int limit = 10)
    {
        var queryEmbedding = await GetEmbeddingAsync(query);
        var vector = new Vector(queryEmbedding);
        
        // Cosine similarity search using pgvector
        return await _db.Products
            .OrderBy(p => p.SearchVector!.CosineDistance(vector))
            .Take(limit)
            .ToListAsync();
    }
}

// EF Core entity with vector
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public string Description { get; set; } = "";
    public Vector? SearchVector { get; set; } // pgvector type
}

// EF Core config
modelBuilder.Entity<Product>()
    .HasIndex(p => p.SearchVector)
    .HasMethod("ivfflat") // approximate nearest neighbor index
    .HasOperators("vector_cosine_ops");
```

---

## ขั้นตอนที่ 895: AI Customer Service Chatbot

```csharp
// AI-powered customer service with context
public class CustomerServiceChatbot
{
    private readonly Kernel _kernel;
    private readonly IOrderRepository _orders;
    private readonly IProductRepository _products;
    
    public async Task<string> RespondAsync(
        Guid customerId, 
        string message,
        List<ChatMessageContent> history)
    {
        // Add tools/plugins for the AI to use
        var orderPlugin = KernelPluginFactory.CreateFromObject(
            new OrderPlugin(_orders, customerId));
        
        _kernel.Plugins.Add(orderPlugin);
        
        // System prompt with context
        history.Insert(0, new ChatMessageContent(AuthorRole.System, $"""
            คุณเป็นผู้ช่วยลูกค้าของ ShopThai ห้างสรรพสินค้าออนไลน์ 
            ตอบเป็นภาษาไทยเสมอ สุภาพ เป็นมิตร และช่วยเหลือ
            
            วันที่ปัจจุบัน: {DateTime.Today:d MMMM yyyy}
            รหัสลูกค้า: {customerId}
            
            คุณสามารถ:
            - ตรวจสอบสถานะคำสั่งซื้อ
            - แนะนำสินค้า
            - ตอบคำถามเกี่ยวกับนโยบายการคืนสินค้า
            """));
        
        history.Add(new ChatMessageContent(AuthorRole.User, message));
        
        var chatService = _kernel.GetRequiredService<IChatCompletionService>();
        var settings = new OpenAIPromptExecutionSettings
        {
            ToolCallBehavior = ToolCallBehavior.AutoInvokeKernelFunctions
        };
        
        var response = await chatService.GetChatMessageContentAsync(
            new ChatHistory(history), settings, _kernel);
        
        history.Add(response);
        return response.ToString();
    }
}

// Plugin with tools for AI
public class OrderPlugin
{
    private readonly IOrderRepository _orders;
    private readonly Guid _customerId;
    
    [KernelFunction, Description("Get order status by order ID")]
    public async Task<string> GetOrderStatusAsync(
        [Description("The order ID to check")] string orderId)
    {
        if (!Guid.TryParse(orderId, out var id)) 
            return "รหัสคำสั่งซื้อไม่ถูกต้อง";
        
        var order = await _orders.GetByIdAsync(id);
        if (order?.CustomerId != _customerId)
            return "ไม่พบคำสั่งซื้อ";
        
        return $"คำสั่งซื้อ {order.Id}: สถานะ {order.Status}, ยอดรวม ฿{order.TotalAmount:N0}";
    }
}
```

---

## ขั้นตอนที่ 896-900: Price Prediction

```csharp
// ML.NET regression for price prediction
public class PricePredictionService
{
    public class ProductFeatures
    {
        public float Category { get; set; }
        public float Brand { get; set; }
        public float Weight { get; set; }
        public float WarrantyMonths { get; set; }
        public float CustomerRating { get; set; }
        public float CompetitorAvgPrice { get; set; }
        [ColumnName("Label")] public float OptimalPrice { get; set; }
    }
    
    public class PricePrediction
    {
        [ColumnName("Score")] public float SuggestedPrice { get; set; }
    }
    
    public void Train(IEnumerable<ProductFeatures> historicalData)
    {
        var data = _ml.Data.LoadFromEnumerable(historicalData);
        
        var features = new[] 
        { 
            "Category", "Brand", "Weight", "WarrantyMonths", 
            "CustomerRating", "CompetitorAvgPrice" 
        };
        
        var pipeline = _ml.Transforms.Concatenate("Features", features)
            .Append(_ml.Regression.Trainers.FastTree(
                labelColumnName: "Label",
                featureColumnName: "Features",
                numberOfLeaves: 20,
                numberOfTrees: 100));
        
        _model = pipeline.Fit(data);
    }
    
    public decimal SuggestPrice(ProductFeatures product)
    {
        var engine = _ml.Model.CreatePredictionEngine<ProductFeatures, PricePrediction>(_model!);
        var prediction = engine.Predict(product);
        return Math.Round((decimal)prediction.SuggestedPrice, 2);
    }
}
```

---

## 📝 สรุป Part 90

| เครื่องมือ | ใช้กับ |
|-----------|--------|
| ML.NET | In-process ML, no external API |
| Semantic Kernel | LLM orchestration (GPT, Claude) |
| pgvector | Semantic/vector search |
| Azure AI Search | Enterprise-grade vector + full-text |

เมื่อใช้ ML vs LLM:
- **ML.NET**: ข้อมูลมี label, predict ตัวเลข, ไม่ต้องการ API
- **LLM**: สร้าง text, เข้าใจ context, คุยกับลูกค้า

---

**ก่อนหน้า → [Part 89: Capstone Event-Driven](part89-capstone-event-driven.md)**  
**ต่อไป → [Part 91: Final Capstone - Complete System Integration](part91-final-integration.md)**
