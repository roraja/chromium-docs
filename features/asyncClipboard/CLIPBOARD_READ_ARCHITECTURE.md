# Clipboard.read() Architecture Overview

This document provides a comprehensive overview of the `clipboard.read()` API architecture in Chromium, showing how clipboard read operations flow through multiple system layers from the Web API down to the platform-specific implementations.

<!-- ## Simplified Component Flow

```mermaid
sequenceDiagram
    participant App as 🌐 Web App
    participant Renderer as 🎭 Blink Renderer
    participant Browser as 🏢 Browser Process
    participant Platform as ⚙️ Platform Layer
    participant OS as 💾 OS Clipboard
    
    Note over App,OS: Clipboard Read Operation Flow
    
    App->>Renderer: navigator.clipboard.read(formats)
    
    Note over Renderer: Security & Permission Checks
    Renderer->>Renderer: Check user gesture
    Renderer->>Renderer: Validate HTTPS context
    Renderer->>Renderer: Request clipboard permission
    
    alt Permission Denied
        Renderer->>App: Promise rejected (SecurityError)
    else Permission Granted
        Renderer->>Browser: IPC: Read clipboard with format filters
        
        Note over Browser: Policy & Validation
        Browser->>Browser: Apply enterprise policies
        Browser->>Browser: Validate MIME types
        
        Browser->>Platform: Get available clipboard types
        Platform->>OS: Query system clipboard
        OS->>Platform: Return available formats
        Platform->>Browser: Filtered format list
        
        Browser->>Platform: Read specific formats
        Platform->>OS: Fetch clipboard data
        OS->>Platform: Raw clipboard data
        Platform->>Browser: Converted data
        
        Note over Browser: Data Processing
        Browser->>Browser: Format conversion
        Browser->>Browser: Create ClipboardItems
        
        Browser->>Renderer: IPC: Return clipboard data
        Renderer->>App: Promise resolved with ClipboardItems
    end
``` -->
## Architecture Diagram

```mermaid
graph TD
    %% Web API Layer
    A["Web Page<br/>navigator.clipboard.read()"] --> B["Clipboard::read()<br/>📁 clipboard.cc:38<br/>• Create promise"]
    
    %% Blink Layer - Permission and Security
    B --> C{"CreateForRead()<br/>📁 clipboard_promise.cc:107<br/>• create promise <br/> HandleRead()"}
    C --> E["HandleRead()<br/>📁 clipboard_promise.cc:245<br/>• Process MIME types<br/>• Validate formats<br/>• Setup IPC call for <br/> permission"]
    
    %% Permission Flow
    E --> EP["HandleReadWithPermission()<br/>📁 clipboard_promise.cc:330<br/>• Check permission status<br/>• Validate user gesture<br/>• Platform permission check"]
    EP -->|Permission Denied| EPR["Promise Rejection<br/>SecurityError<br/>📁 clipboard_promise.cc"]
    EP -->|Permission Granted| ERF["ReadAvailableCustomAndStandardFormats()<br/>📁 system_clipboard.cc<br/>• Query OS clipboard<br/>• Get format list"]
    
    %% Format Discovery
    ERF -->  G["ReadAvailableTypes()<br/>📁 clipboard_host_impl.cc:166<br/>• Receive MIME filters<br/>• Coordinate platform<br/>• Apply policies"]
    
    %% Security and Data Processing
    G --> H{"Data Validation<br/>📁 clipboard_host_impl.cc<br/>• Check MIME types<br/>• Security policies<br/>• Format support"}
    H --> I["Platform Access<br/>• Route to OS impl<br/>• Apply MIME filter<br/>• Format detection"]
    
    %% Platform Abstraction
    I --> J{"Platform Detection<br/>• Windows: ClipboardWin<br/>• Mac: ClipboardMac<br/>• Linux: ClipboardOzone<br/>• Android: ClipboardAndroid"}
    
    %% Platform-Specific Implementations
    J -->|Windows| K["ReadAvailableTypes()<br/>📁 clipboard_win.cc:372<br/>• GetStandardFormats()<br/>• Filter by MIME"]
    J -->|Mac| L["ReadAvailableTypes()<br/>📁 clipboard_mac.mm<br/>• Query NSPasteboard<br/>• Map MIME types"]
    J -->|Linux| M["ReadAvailableTypes()<br/>📁 clipboard_ozone.cc<br/>• X11/Wayland<br/>• MIME negotiation"]
    J -->|Android| N["ReadAvailableTypes()<br/>📁 clipboard_android.cc<br/>• ClipboardManager<br/>• Limited formats"]
    
    %% Platform System APIs
    K --> S["Raw Platform Data<br/>• CF_HTML, CF_TEXT<br/>• Format parsing<br/>• Encoding detect"]
    L --> S
    M --> S
    N --> S
    S --> ERAF["OnReadAvailableFormatNames()<br/>📁 clipboard_promise.cc:407<br/>• Receive format list<br/>• Filter supported formats<br/>• Initialize clipboard_item_data_"]
    ERAF --> AC
    U --> AB["OnRead() Callback<br/>📁 clipboard_promise.cc<br/>• Store Blob result<br/>• Increment index<br/>• Check completion"]
    AB --> AC{"ReadNextRepresentation Loop<br/>📁 clipboard_promise.cc<br/>• clipboard_representation_index++<br/>• More formats to read?"}
    AC -->|More formats| AD["ClipboardReader::Create()<br/>📁 clipboard_promise.cc<br/>• Create reader for next format<br/>• Continue iteration"]
    AD --> U["Data Packaging<br/>📁 clipboard_reader.cc<br/>• ReadNextData()<br/>• Create ClipboardItem<br/>• Apply priorities"]
    AC -->|All complete| AE["ResolveRead()<br/>📁 clipboard_promise.cc<br/>• Combine all format data<br/>• Create ClipboardItem array"]
    AE --> V["Mojo Response<br/>📁 clipboard_host_impl.cc<br/>• Send results<br/>• Respect MIME prefs"]
    V --> W["Promise Resolution<br/>📁 clipboard_promise.cc<br/>• Package items<br/>• Return to JS"]
    W --> X["JavaScript Result<br/>• getType('text/html')<br/>• MIME-filtered data<br/>• Blob/String ready"]
    
    %% Error Handling Path
    H -->|Invalid| Y["Error Response<br/>📁 clipboard_promise.cc<br/>• Unsupported MIME<br/>• No formats"]
    Y --> Z["Promise Rejection<br/>📁 clipboard_promise.cc<br/>• DOMException<br/>• NotAllowedError"]
    
    
    %% Security Annotations
    classDef webLayer fill:#e1f5fe,stroke:#01579b,stroke-width:2px,color:#000000
    classDef blinkLayer fill:#f3e5f5,stroke:#4a148c,stroke-width:2px,color:#000000
    classDef ipcLayer fill:#fff3e0,stroke:#e65100,stroke-width:2px,color:#000000
    classDef browserLayer fill:#e8f5e8,stroke:#1b5e20,stroke-width:2px,color:#000000
    classDef platformLayer fill:#fce4ec,stroke:#880e4f,stroke-width:2px,color:#000000
    classDef systemLayer fill:#f1f8e9,stroke:#33691e,stroke-width:2px,color:#000000
    classDef mimeLayer fill:#fff9c4,stroke:#f57f17,stroke-width:3px,color:#000000
    
    class A webLayer
    class B,C,E,EP,EPR,ERF,ERAF blinkLayer
    class F ipcLayer
    class G,H,I,V browserLayer
    class J,K,L,M,N,T,U,AB,AC,AD,AE platformLayer
    class O,P,Q,R,S systemLayer
    class AA mimeLayer
