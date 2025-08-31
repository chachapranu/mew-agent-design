
# Gemini Assistant - High-Level Architecture

```mermaid
graph TD
    subgraph User_Interface["User Interface"]
        UI[Samsung Family Hub UI / Voice Input]
    end

    subgraph Core_Logic["Core Logic (Semantic Kernel)"]
        Orchestrator[Orchestrator Agent]
        SK_Core["Semantic Kernel Core\n(Planner, Functions, Memory)"]
        Orchestrator -- Uses --> SK_Core
    end

    subgraph Specialized_Agents["Specialized Agents (Plugins)"]
        PIM_Agent["PIM Agent (Calendar, Notes, Contacts)"]
        Entertainment_Agent["Entertainment Agent\n(Music, TV, Podcasts)"]
        Recipe_Food_Agent["Recipe & Food Agent\n(Fridge, Recipes, Lists)"]
        Device_Control_Agent["Device Control Agent\n(Volume, Temp, Settings)"]
        Cross_Device_Agent["Cross-Device Agent\n(Handoff, Multi-device Scenes)"]
        Web_Search_Agent[Web Search Agent]
    end

    subgraph Service_Data["Service & Data Layer"]
        Tizen_MCP[Tizen MCP Server]
        Cloud_Services["Cloud Services\n(Weather, Search, Music APIs, LLMs)"]
        UserDataStore["User Data Store\n(Preferences, Routines, History)"]
    end

    subgraph Platform_Hardware["Platform & Hardware"]
        Device_APIs["Device APIs\n(Fridge Cam, Temp Sensor)"]
        Platform_Services[Tizen Platform Services]
        BT_Speaker[Bluetooth Speaker]
        TV[Samsung TV]
    end

    %% Connections
    UI -- Raw Input --> Orchestrator
    Orchestrator -- Delegates to --> PIM_Agent
    Orchestrator -- Delegates to --> Entertainment_Agent
    Orchestrator -- Delegates to --> Recipe_Food_Agent
    Orchestrator -- Delegates to --> Device_Control_Agent
    Orchestrator -- Delegates to --> Cross_Device_Agent
    Orchestrator -- Delegates to --> Web_Search_Agent

    Device_Control_Agent -- Issues Commands --> Tizen_MCP
    Recipe_Food_Agent -- Accesses --> Tizen_MCP
    Tizen_MCP -- Controls --> Device_APIs
    Tizen_MCP -- Controls --> Platform_Services

    PIM_Agent -- Accesses --> UserDataStore
    PIM_Agent -- Accesses --> Cloud_Services
    Entertainment_Agent -- Accesses --> Cloud_Services
    Recipe_Food_Agent -- Accesses --> Cloud_Services
    Web_Search_Agent -- Accesses --> Cloud_Services

    Cross_Device_Agent -- Coordinates --> TV
    Cross_Device_Agent -- Coordinates --> BT_Speaker
```

### Component Descriptions

*   **Orchestrator Agent:** The brain of the system. It receives all user input. It uses the **Semantic Kernel Planner** to understand the user's intent, break down complex requests into steps, and delegate those steps to the appropriate specialized agent. For "Suggest a recipe based on what's in my fridge," it would first call the `Recipe_Food_Agent` to get inventory, then call it again to find recipes.
*   **Specialized Agents:** These are essentially "Plugins" in the Semantic Kernel terminology. Each is a self-contained set of functions for a specific domain.
    *   **Device Control Agent:** The primary client of the Tizen MCP. It handles all direct hardware interactions like `setVolume()`, `getTemperature()`, `setLight()`.
    *   **Recipe & Food Agent:** Manages fridge inventory (via MCP's camera API), searches for recipes (via a Cloud Service), and manages shopping lists (in the User Data Store).
    *   **PIM Agent:** Handles Personal Information Management. Connects to calendar/contact APIs (like Google/Outlook) and manages user-specific notes.
    *   **Cross-Device Agent:** Manages interactions that span multiple devices, handling discovery, connection, and control (e.g., "Play this on the living room TV").
*   **Tizen MCP Server:** This is the crucial abstraction layer provided by the platform. It exposes a secure and stable API for all hardware and low-level Tizen services. Our agents treat this as a single, reliable endpoint.
*   **User Data Store:** A database that stores user preferences, learned routines, conversation history, and other personalization data. This is the key to making the assistant truly personal and proactive.

