stateDiagram-v2
    [*] --> Idle
    
    Idle --> IntentRecognition: User Request
    
    IntentRecognition --> SingleDevice: Simple Command
    IntentRecognition --> MultiDevice: Complex Scenario
    
    SingleDevice --> ExecuteLocal: Device Available
    SingleDevice --> ErrorHandling: Device Unavailable
    
    MultiDevice --> PlanGeneration: Analyze Dependencies
    
    PlanGeneration --> ResourceCheck: Check All Devices
    
    ResourceCheck --> Scheduling: All Available
    ResourceCheck --> Adaptation: Some Unavailable
    
    Adaptation --> AlternativePlan: Find Alternatives
    Adaptation --> PartialExecution: Execute Possible Steps
    
    Scheduling --> ParallelExecution: Independent Tasks
    Scheduling --> SequentialExecution: Dependent Tasks
    
    ParallelExecution --> Synchronization
    SequentialExecution --> Synchronization
    
    Synchronization --> ResultAggregation
    
    ResultAggregation --> Success: All Complete
    ResultAggregation --> PartialSuccess: Some Complete
    
    ExecuteLocal --> Success
    AlternativePlan --> Success
    PartialExecution --> PartialSuccess
    
    Success --> UpdateContext
    PartialSuccess --> UpdateContext
    ErrorHandling --> UpdateContext
    
    UpdateContext --> Learning: Store Patterns
    Learning --> Idle
    
    note right of MultiDevice
        Example: "Bedtime routine"
        - Lock doors (Smart Lock)
        - Dim lights (Smart Bulbs)  
        - Set alarm (Phone)
        - Start dishwasher (Appliance)
        - Lower thermostat (HVAC)
    end note
    
    note right of Adaptation
        Graceful degradation when
        devices are offline or
        unavailable
    end note