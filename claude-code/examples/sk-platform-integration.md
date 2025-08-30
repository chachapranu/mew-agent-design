# Semantic Kernel Platform Integration Examples

## Direct Platform API Usage with SK Functions

### 1. Enhanced Kitchen Assistant with Platform APIs

```csharp
using Microsoft.SemanticKernel;
using Microsoft.SemanticKernel.Connectors.OpenAI;

public class EnhancedKitchenAssistant
{
    private readonly IKernel _kernel;
    
    public EnhancedKitchenAssistant()
    {
        _kernel = BuildKernel();
    }
    
    private IKernel BuildKernel()
    {
        var builder = Kernel.CreateBuilder();
        
        // Add AI services
        builder.AddAzureOpenAIChatCompletion(
            "gpt-4",
            endpoint,
            apiKey);
        
        // Add platform native functions
        builder.Plugins.AddFromType<FridgeFunctions>();
        builder.Plugins.AddFromType<OvenFunctions>();
        builder.Plugins.AddFromType<DishwasherFunctions>();
        builder.Plugins.AddFromType<SmartThingsFunctions>();
        
        // Add semantic functions from prompts
        builder.Plugins.AddFromPromptDirectory("./Prompts/Kitchen");
        
        return builder.Build();
    }
    
    // Complex orchestration combining LLM + Platform APIs
    public async Task<MealPrepPlan> PrepareMealWithAssistance(string mealRequest)
    {
        // Step 1: Use LLM to understand request
        var mealIntent = await _kernel.InvokePromptAsync(
            "Understand this meal request and extract: cuisine type, serving size, dietary restrictions: {{$input}}",
            new() { ["input"] = mealRequest });
        
        // Step 2: Check fridge inventory using native function
        var inventory = await _kernel.InvokeAsync<FridgeInventory>(
            "FridgeFunctions",
            "GetDetailedInventory");
        
        // Step 3: Generate recipe with LLM based on real inventory
        var recipePrompt = $@"Create a recipe for:
        Intent: {mealIntent}
        Available ingredients: {JsonSerializer.Serialize(inventory)}
        Output format: JSON with ingredients, steps, timings";
        
        var recipe = await _kernel.InvokePromptAsync<Recipe>(recipePrompt);
        
        // Step 4: Prepare appliances using native functions
        var prepTasks = new List<Task>();
        
        if (recipe.RequiresOven)
        {
            prepTasks.Add(_kernel.InvokeAsync(
                "OvenFunctions",
                "PreheatOven",
                new() { ["temperature"] = recipe.OvenTemp }));
        }
        
        if (recipe.RequiresStovetop)
        {
            prepTasks.Add(_kernel.InvokeAsync(
                "SmartThingsFunctions",
                "SetInductionPower",
                new() { ["zone"] = "front", ["power"] = recipe.StovePower }));
        }
        
        await Task.WhenAll(prepTasks);
        
        // Step 5: Create step-by-step guidance with timers
        var plan = new MealPrepPlan
        {
            Recipe = recipe,
            Steps = await GenerateGuidanceSteps(recipe),
            ApplianceStatus = await GetApplianceStatus()
        };
        
        return plan;
    }
}

// Native Platform Functions
public class FridgeFunctions
{
    private readonly ITizenMCPClient _mcpClient;
    private readonly ILocalMLService _mlService;
    
    [KernelFunction("GetDetailedInventory")]
    [Description("Gets complete fridge inventory with quantities and freshness")]
    public async Task<FridgeInventory> GetDetailedInventory()
    {
        // Get raw sensor data
        var sensorData = await _mcpClient.ExecuteAsync(new MCPRequest
        {
            Resource = "fridge.sensors",
            Action = "getAllSensors"
        });
        
        // Get camera image for CV analysis
        var image = await _mcpClient.ExecuteAsync(new MCPRequest
        {
            Resource = "fridge.camera",
            Action = "captureImage"
        });
        
        // Use local ML for detection
        var detected = await _mlService.Vision.DetectItems(image.Data);
        
        // Combine sensor + vision data
        return new FridgeInventory
        {
            Items = MergeDetections(sensorData.Data, detected),
            Temperature = sensorData.Data.temperature,
            Humidity = sensorData.Data.humidity,
            LastUpdated = DateTime.UtcNow
        };
    }
    
    [KernelFunction("MonitorFreshness")]
    [Description("Monitors food freshness and sends alerts")]
    public async Task<FreshnessReport> MonitorFreshness()
    {
        var inventory = await GetDetailedInventory();
        var report = new FreshnessReport();
        
        foreach (var item in inventory.Items)
        {
            var freshness = await _mlService.Vision.AssessFreshness(
                item.Image,
                item.Type);
            
            if (freshness.DaysRemaining <= 2)
            {
                report.ExpiringSoon.Add(item);
            }
            
            item.FreshnessScore = freshness.Score;
        }
        
        return report;
    }
}

public class OvenFunctions
{
    private readonly ITizenMCPClient _mcpClient;
    
    [KernelFunction("PreheatOven")]
    [Description("Preheats oven to specified temperature")]
    public async Task<OvenStatus> PreheatOven(
        [Description("Temperature in Celsius")] int temperature,
        [Description("Mode: bake, broil, convection")] string mode = "bake")
    {
        var response = await _mcpClient.ExecuteAsync(new MCPRequest
        {
            Resource = "oven.control",
            Action = "preheat",
            Parameters = new
            {
                temperature = temperature,
                mode = mode,
                notification = true
            }
        });
        
        return new OvenStatus
        {
            IsPreheating = true,
            TargetTemp = temperature,
            CurrentTemp = response.Data.currentTemp,
            EstimatedTime = response.Data.estimatedTime
        };
    }
    
    [KernelFunction("SmartCook")]
    [Description("Uses AI to monitor and adjust cooking")]
    public async Task<CookingSession> SmartCook(
        [Description("Type of food")] string foodType,
        [Description("Desired doneness")] string doneness = "medium")
    {
        // Start AI cooking mode
        var session = await _mcpClient.ExecuteAsync(new MCPRequest
        {
            Resource = "oven.ai",
            Action = "startSmartCook",
            Parameters = new
            {
                foodType = foodType,
                doneness = doneness,
                useCamera = true,
                autoAdjust = true
            }
        });
        
        return new CookingSession
        {
            SessionId = session.Data.sessionId,
            EstimatedTime = session.Data.estimatedTime,
            MonitoringEnabled = true
        };
    }
}
```

### 2. Advanced Multi-Agent Orchestration

```csharp
public class MultiAgentOrchestrator
{
    private readonly IKernel _kernel;
    private readonly IAgentRuntime _agentRuntime;
    
    public async Task<DinnerPartyPlan> PlanDinnerParty(DinnerPartyRequest request)
    {
        // Create specialized agents with SK functions
        var agents = new List<IAgent>
        {
            new ChefAgent(_kernel),
            new HomeControlAgent(_kernel),
            new EntertainmentAgent(_kernel),
            new ShoppingAgent(_kernel)
        };
        
        // Create agent group for collaboration
        var agentGroup = new AgentGroupChat(agents)
        {
            ExecutionSettings = new()
            {
                TerminationStrategy = new DinnerPartyCompletionStrategy(),
                SelectionStrategy = new SequentialSelectionStrategy()
            }
        };
        
        // Initial prompt to start planning
        var initialMessage = new ChatMessageContent(
            AuthorRole.User,
            $"Plan a dinner party for {request.Guests} guests, {request.Date}, preferences: {request.Preferences}");
        
        agentGroup.AddChatMessage(initialMessage);
        
        // Let agents collaborate
        await foreach (var message in agentGroup.InvokeAsync())
        {
            Console.WriteLine($"{message.AuthorName}: {message.Content}");
            
            // Agents will use their SK functions to:
            // - Check inventory and suggest menu (ChefAgent)
            // - Order missing items (ShoppingAgent)  
            // - Set home ambiance (HomeControlAgent)
            // - Prepare entertainment (EntertainmentAgent)
        }
        
        return ExtractPlan(agentGroup.ChatHistory);
    }
}

public class ChefAgent : IAgent
{
    private readonly IKernel _kernel;
    
    public ChefAgent(IKernel kernel)
    {
        _kernel = kernel;
        Name = "Chef";
        Description = "Handles meal planning and cooking";
        
        // Add chef-specific plugins
        _kernel.Plugins.Add(KernelPluginFactory.CreateFromType<RecipeFunctions>());
        _kernel.Plugins.Add(KernelPluginFactory.CreateFromType<NutritionFunctions>());
    }
    
    public async Task<ChatMessageContent> InvokeAsync(ChatHistory history)
    {
        var lastMessage = history.Last();
        
        if (lastMessage.Content.Contains("menu") || lastMessage.Content.Contains("food"))
        {
            // Check what we can make
            var inventory = await _kernel.InvokeAsync<FridgeInventory>(
                "FridgeFunctions",
                "GetDetailedInventory");
            
            // Generate menu suggestions
            var menuPrompt = $@"Based on inventory: {inventory}
            Create an elegant dinner menu for the party.
            Consider dietary restrictions and seasonal ingredients.";
            
            var menu = await _kernel.InvokePromptAsync(menuPrompt);
            
            // Check nutrition balance
            var nutrition = await _kernel.InvokeAsync(
                "NutritionFunctions",
                "AnalyzeMenu",
                new() { ["menu"] = menu });
            
            return new ChatMessageContent(
                AuthorRole.Assistant,
                $"I've created a menu: {menu}\nNutrition analysis: {nutrition}",
                metadata: new Dictionary<string, object> { ["menu"] = menu });
        }
        
        return null;
    }
}
```

### 3. Context-Aware Platform Integration

```csharp
public class ContextAwarePlatformService
{
    private readonly IKernel _kernel;
    private readonly IMemoryStore _memoryStore;
    
    public async Task<SmartResponse> HandleContextualRequest(string userInput, UserContext context)
    {
        // Enrich kernel with context
        var contextualKernel = _kernel.Clone();
        
        // Add user preferences to kernel context
        contextualKernel.Data["userPreferences"] = context.Preferences;
        contextualKernel.Data["familyMembers"] = context.Family;
        contextualKernel.Data["deviceStates"] = await GetAllDeviceStates();
        
        // Create context-aware prompt
        var enhancedPrompt = $@"
        User: {userInput}
        Time: {DateTime.Now}
        Location: Kitchen
        Recent activity: {context.RecentActivity}
        Family home: {string.Join(", ", context.FamilyMembersHome)}
        
        Provide a helpful response considering the context.
        Use available functions to check device states and control them if needed.";
        
        // Process with full context
        var plan = await contextualKernel.CreatePlanAsync(enhancedPrompt);
        
        // Execute plan with platform functions
        var result = await plan.InvokeAsync(contextualKernel);
        
        // Store interaction in memory for learning
        await _memoryStore.SaveInteractionAsync(new Interaction
        {
            Input = userInput,
            Context = context,
            Response = result,
            Timestamp = DateTime.UtcNow,
            Satisfaction = await GatherFeedback()
        });
        
        return result;
    }
    
    // Platform state aggregation
    private async Task<DeviceStates> GetAllDeviceStates()
    {
        var tasks = new Dictionary<string, Task<object>>();
        
        // Parallel platform queries
        tasks["fridge"] = _kernel.InvokeAsync<object>("FridgeFunctions", "GetStatus");
        tasks["oven"] = _kernel.InvokeAsync<object>("OvenFunctions", "GetStatus");
        tasks["dishwasher"] = _kernel.InvokeAsync<object>("DishwasherFunctions", "GetStatus");
        tasks["climate"] = _kernel.InvokeAsync<object>("ClimateControl", "GetStatus");
        tasks["lighting"] = _kernel.InvokeAsync<object>("SmartLighting", "GetStatus");
        
        await Task.WhenAll(tasks.Values);
        
        return new DeviceStates
        {
            Devices = tasks.ToDictionary(
                kvp => kvp.Key,
                kvp => kvp.Value.Result)
        };
    }
}
```

### 4. Reactive Platform Event Handling

```csharp
public class ReactivePlatformService
{
    private readonly IKernel _kernel;
    private readonly ISubject<PlatformEvent> _eventStream;
    
    public ReactivePlatformService()
    {
        _eventStream = new Subject<PlatformEvent>();
        SetupReactiveChains();
    }
    
    private void SetupReactiveChains()
    {
        // Fridge door open too long
        _eventStream
            .Where(e => e.Type == "fridge.door.open")
            .Where(e => e.Duration > TimeSpan.FromMinutes(1))
            .Subscribe(async e =>
            {
                // Use SK to generate contextual reminder
                var reminder = await _kernel.InvokePromptAsync(
                    "Generate a friendly reminder that the fridge door is open. Be helpful about what they might be looking for.");
                
                await _kernel.InvokeAsync(
                    "AudioFunctions",
                    "Speak",
                    new() { ["text"] = reminder });
            });
        
        // Cooking timer expired
        _eventStream
            .Where(e => e.Type == "timer.expired")
            .Subscribe(async e =>
            {
                var nextStep = await _kernel.InvokeAsync(
                    "RecipeFunctions",
                    "GetNextStep",
                    new() { ["timerId"] = e.Data.TimerId });
                
                await _kernel.InvokeAsync(
                    "DisplayFunctions",
                    "ShowNotification",
                    new() { ["content"] = nextStep });
            });
        
        // Energy usage spike
        _eventStream
            .Where(e => e.Type == "energy.spike")
            .Subscribe(async e =>
            {
                // Analyze and optimize
                var analysis = await _kernel.InvokePromptAsync(
                    $"Energy spike detected: {e.Data}. Analyze and suggest optimizations.");
                
                var optimization = await _kernel.CreatePlanAsync(analysis.ToString());
                await optimization.InvokeAsync(_kernel);
            });
    }
}
```

### 5. Semantic Functions with Platform Integration

```yaml
# Prompts/Kitchen/SmartCooking.yaml
name: SmartCooking
description: Intelligent cooking assistance with appliance control
template: |
  You are a professional chef assistant with access to smart kitchen appliances.
  
  Current cooking session:
  Recipe: {{$recipe}}
  Step: {{$currentStep}}
  Elapsed time: {{$elapsedTime}}
  
  Appliance states:
  Oven: {{$ovenStatus}}
  Stovetop: {{$stovetopStatus}}
  
  The user says: {{$input}}
  
  Provide cooking guidance and use these functions if needed:
  - AdjustOvenTemp: Modify oven temperature
  - SetTimer: Create a cooking timer
  - ShowTechnique: Display cooking technique video
  - CheckDoneness: Use AI vision to check food doneness
  
  Response format:
  1. Advice or answer
  2. Any appliance adjustments needed
  3. Next step preview

# Prompts/Energy/OptimizeUsage.yaml  
name: OptimizeEnergyUsage
description: Analyzes and optimizes home energy usage
template: |
  Analyze current energy usage and create optimization plan:
  
  Current usage by device:
  {{$deviceUsage}}
  
  Peak hours: {{$peakHours}}
  Current rate: {{$currentRate}}
  Weather forecast: {{$weather}}
  Family schedule: {{$schedule}}
  
  Available control functions:
  - SetThermostat: Adjust temperature
  - ScheduleAppliance: Delay appliance start
  - DimLights: Adjust lighting levels
  - SetEcoMode: Enable eco mode on devices
  
  Create an optimization plan that:
  1. Reduces peak usage
  2. Maintains comfort
  3. Considers family schedule
  4. Estimates savings
```

### 6. Advanced Platform Function Chaining

```csharp
public class PlatformFunctionChains
{
    private readonly IKernel _kernel;
    
    [KernelFunction("MorningRoutine")]
    [Description("Executes complete morning routine")]
    public async Task<RoutineResult> ExecuteMorningRoutine(string userId)
    {
        var pipeline = _kernel.CreateFunctionPipeline(
            "UserProfile.GetPreferences",
            "Weather.GetForecast", 
            "Calendar.GetTodaySchedule",
            "Coffee.StartBrewing",
            "Climate.AdjustForComfort",
            "News.GetPersonalizedBriefing",
            "Kitchen.SuggestBreakfast",
            "Display.ShowMorningDashboard"
        );
        
        var context = new KernelArguments
        {
            ["userId"] = userId,
            ["time"] = DateTime.Now
        };
        
        var result = await pipeline.InvokeAsync(_kernel, context);
        
        return new RoutineResult
        {
            ExecutedSteps = pipeline.Functions.Count,
            Duration = result.Metadata["duration"],
            Status = "Complete"
        };
    }
}
```

## Performance Optimizations

```csharp
public class OptimizedPlatformService
{
    private readonly IKernel _kernel;
    private readonly IMemoryCache _cache;
    
    public async Task<T> CachedPlatformCall<T>(string function, object parameters, TimeSpan? ttl = null)
    {
        var cacheKey = $"{function}:{JsonSerializer.Serialize(parameters)}";
        
        if (_cache.TryGetValue<T>(cacheKey, out var cached))
        {
            return cached;
        }
        
        var result = await _kernel.InvokeAsync<T>(function, parameters);
        
        _cache.Set(cacheKey, result, ttl ?? TimeSpan.FromMinutes(5));
        
        return result;
    }
    
    // Batch platform operations
    public async Task<Dictionary<string, object>> BatchPlatformCalls(
        Dictionary<string, PlatformCall> calls)
    {
        var tasks = calls.ToDictionary(
            kvp => kvp.Key,
            kvp => _kernel.InvokeAsync(kvp.Value.Function, kvp.Value.Parameters)
        );
        
        await Task.WhenAll(tasks.Values);
        
        return tasks.ToDictionary(
            kvp => kvp.Key,
            kvp => kvp.Value.Result
        );
    }
}
```