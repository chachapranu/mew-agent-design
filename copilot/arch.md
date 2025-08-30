flowchart LR
  subgraph UI
    U[User] --> UI[Chat & Voice Interface]
    UI --> ML[Multimodal Renderer]
  end

  UI --> NLU[NLU Service]
  NLU --> ORC[Magentic Manager]

  subgraph Agents
    ORC --> A1[ConversationAgent]
    ORC --> A2[CalendarAgent]
    ORC --> A3[RecipeAgent]
    ORC --> A4[MusicAgent]
    ORC --> A5[TVAgent]
    ORC --> A6[SettingsAgent]
    ORC --> A7[InventoryAgent]
    ORC --> A8[MaintenanceAgent]
  end

  subgraph SK["Semantic Kernel"]
    SK[Kernal Host]
    SK --> Services[AI & Logging Services]
    SK --> Plugins[Domain & Utility Plugins]
  end

  Agents --> SK
  SK --> Connectors[Device Connector Layer]

  subgraph Devices
    Connectors --> MCP[Tizen MCP Server]
    MCP --> Fridge[Fridge]
    MCP --> TV[TV]
    MCP --> Speaker[Bluetooth Speaker]
    MCP --> HVAC[Climate Control]
    MCP --> Washer[Washer]
  end

  SK --> Data[(Session & Profile Store)]
  ORC --> Data
  NLU --> Data