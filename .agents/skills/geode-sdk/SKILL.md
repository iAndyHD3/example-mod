---
name: geode-sdk
description: Always use this skill when helping users write Geometry Dash mods using the Geode SDK. This includes explaining the $modify syntax, hooking patterns, creating UI elements, managing fields, working with events, layouts, settings, and all other Geode-specific coding patterns. Trigger whenever the user mentions Geode SDK, Geometry Dash modding, or asks for help with GD mod development.
---

# Geode SDK - Coding Patterns & Tutorials

## What is Geode SDK?

Geode is a Geometry Dash mod loader and modding SDK with a modern approach towards mod development. It provides:
- A clean, readable hooking syntax with `$modify`
- Cross-platform compatibility (Windows, macOS, Android, iOS)
- Built-in UI components and utilities
- Mod incompatibility prevention
- Easy development with managed hooks and dependencies

## Core Concepts

### 1. The `$modify` Macro - The Heart of Geode

The `$modify` macro is Geode's primary hooking mechanism. It allows you to override or extend functions in Geometry Dash classes.

#### Basic Syntax:
```cpp
#include <Geode/modify/ClassName.hpp>

class $modify(ClassName) {
    // Override or extend functions here
};
```

#### Naming Your Modified Class:
When you need to reference your modified class (e.g., for button callbacks), provide a custom name:

```cpp
class $modify(MyModName, MenuLayer) {
    void onMyButton(CCObject* sender) {
        // Your code
    }
    
    bool init() {
        if (!MenuLayer::init()) return false;
        
        auto btn = CCMenuItemSpriteExtra::create(
            sprite,
            this,
            menu_selector(MyModName::onMyButton)
        );
        // ...
        return true;
    }
};
```

### 2. Hooking Patterns

#### Pattern A: Extending Original Behavior (Most Common)
```cpp
class $modify(MenuLayer) {
    bool init() {
        // ALWAYS call the original first
        if (!MenuLayer::init()) return false;
        
        // Add your custom code here
        auto label = CCLabelBMFont::create("Hello, World!", "bigFont.fnt");
        label->setPosition(100, 100);
        this->addChild(label);
        
        return true;
    }
};
```

**Key Pattern:** Always check if the original function returns false (indicates failure). This is critical for `init()` functions.

#### Pattern B: Replacing Original Behavior
```cpp
class $modify(MenuLayer) {
    void onMoreGames(CCObject* target) {
        // Completely replace the original behavior
        FLAlertLayer::create(
            "Geode",
            "Hello World from my Custom Mod!",
            "OK"
        )->show();
        // Original function NOT called
    }
};
```

#### Pattern C: Wrapping Original Behavior
```cpp
class $modify(PlayerObject) {
    void pushButton(PlayerButton button) {
        log::info("Before jump");
        
        // Call original
        PlayerObject::pushButton(button);
        
        log::info("After jump");
    }
};
```

### 3. Fields - Adding Member Variables to Hooked Classes

Fields allow you to add member variables to existing GD classes seamlessly:

```cpp
class $modify(PlayerObject) {
    struct Fields {
        int m_jumpCount = 0;
        std::string m_playerName = "Default";
        
        // Constructor
        Fields() {
            log::debug("Fields initialized!");
        }
        
        // Destructor
        ~Fields() {
            log::debug("Fields destroyed!");
        }
    };
    
    void pushButton(PlayerButton button) {
        m_fields->m_jumpCount++;
        log::info("Player jumped {} times", m_fields->m_jumpCount);
        
        PlayerObject::pushButton(button);
    }
};
```

**Accessing Fields from Outside:**
```cpp
// Use static_cast, NOT typeinfo_cast
static_cast<MyModClass*>(gameObject)->m_fields->m_myField = 42;
```

**Key Points:**
- Access fields with `m_fields->`
- Fields are automatically constructed and destructed
- Perfect for tracking state per instance

### 4. Creating Buttons

#### Basic Button Pattern:
```cpp
class $modify(MyMod, MenuLayer) {
    bool init() {
        if (!MenuLayer::init()) return false;
        
        // 1. Create button sprite
        auto sprite = ButtonSprite::create("Click Me!");
        // Or use existing GD sprites:
        // auto sprite = CCSprite::createWithSpriteFrameName("GJ_playBtn_001.png");
        
        // 2. Create button with callback
        auto button = CCMenuItemSpriteExtra::create(
            sprite,
            this,
            menu_selector(MyMod::onMyButton)
        );
        
        // 3. Create or get menu
        auto menu = CCMenu::create();
        menu->addChild(button);
        menu->setPosition(100, 100);
        
        // 4. Add menu to layer
        this->addChild(menu);
        
        return true;
    }
    
    void onMyButton(CCObject* sender) {
        log::info("Button clicked!");
        
        // Get the button that was clicked:
        auto btn = static_cast<CCMenuItemSpriteExtra*>(sender);
    }
};
```

#### Button with Multiple Callbacks (Using Tags):
```cpp
class $modify(MyMod, MenuLayer) {
    struct Fields {
        int m_counter = 0;
    };
    
    bool init() {
        if (!MenuLayer::init()) return false;
        
        auto menu = CCMenu::create();
        
        auto btn1 = CCMenuItemSpriteExtra::create(
            ButtonSprite::create("Increment"),
            this,
            menu_selector(MyMod::onCounterButton)
        );
        btn1->setTag(1);  // Tag to differentiate
        
        auto btn2 = CCMenuItemSpriteExtra::create(
            ButtonSprite::create("Decrement"),
            this,
            menu_selector(MyMod::onCounterButton)
        );
        btn2->setTag(-1);
        
        menu->addChild(btn1);
        menu->addChild(btn2);
        this->addChild(menu);
        
        return true;
    }
    
    void onCounterButton(CCObject* sender) {
        m_fields->m_counter += sender->getTag();
        log::info("Counter: {}", m_fields->m_counter);
    }
};
```

### 5. Layouts - Automatic Positioning

Layouts eliminate manual positioning headaches. Geode uses `AxisLayout` for all layout types.

#### Using Existing Layouts:
```cpp
class $modify(MenuLayer) {
    bool init() {
        if (!MenuLayer::init()) return false;
        
        // Get an existing menu with layout
        auto menu = this->getChildByID("bottom-menu");
        
        // Add your button
        auto btn = CCMenuItemSpriteExtra::create(/* ... */);
        menu->addChild(btn);
        
        // Update layout to reposition all children
        menu->updateLayout();
        
        return true;
    }
};
```

#### Creating Your Own Layout:
```cpp
auto menu = CCMenu::create();

// Set layout type
menu->setLayout(
    RowLayout::create()
        ->setGap(10.0f)           // Space between items
        ->setAxisAlignment(AxisAlignment::Start)
);

// Add children
menu->addChild(button1);
menu->addChild(button2);
menu->addChild(button3);

// Apply layout
menu->updateLayout();
```

**Layout Types:**
- `RowLayout::create()` - Horizontal row
- `ColumnLayout::create()` - Vertical column
- `AxisLayout::create()` - General purpose (can do rows, columns, grids)

### 6. Events - Communication Between Mods

Events allow mods to communicate without direct dependencies.

#### Listening for Events Globally:
```cpp
#include <Geode/loader/Event.hpp>

// In a global scope or $execute block
$execute {
    new EventListener<EventFilter<MyCustomEvent>>(
        +[](MyCustomEvent* event) {
            log::info("Event received: {}", event->getData());
            
            // Stop propagation or continue
            return ListenerResult::Propagate;
        }
    );
}
```

#### Listening for Events on Nodes:
```cpp
auto myNode = CCNode::create();

myNode->addEventListener<SomeEventFilter>(
    [](EventType* event) {
        // Handle event
        return ListenerResult::Propagate;
    },
    // Additional filter constructor args...
);
```

#### Creating Custom Events:
```cpp
class MyCustomEvent : public Event {
protected:
    std::string m_data;
    
public:
    MyCustomEvent(std::string data) : m_data(data) {}
    std::string getData() const { return m_data; }
};

// Post the event
MyCustomEvent("some data").post();
```

### 7. Settings - User Configuration

Settings are defined in `mod.json` and accessed in code:

#### Accessing Settings:
```cpp
auto boolValue = Mod::get()->getSettingValue<bool>("my-bool-setting");
auto intValue = Mod::get()->getSettingValue<int64_t>("my-int-setting");
auto floatValue = Mod::get()->getSettingValue<double>("my-float-setting");
auto strValue = Mod::get()->getSettingValue<std::string>("my-string-setting");
```

#### Listening for Setting Changes:
```cpp
$execute {
    listenForSettingChanges("my-setting", [](double value) {
        log::info("Setting changed to: {}", value);
        // React to setting change
    });
}
```

### 8. Creating Popups

#### Simple Info Popup:
```cpp
FLAlertLayer::create(
    "Title",
    "Message content here!",
    "OK"
)->show();
```

#### Confirmation Popup:
```cpp
geode::createQuickPopup(
    "Confirm Action",
    "Are you sure?",
    "Cancel", "Confirm",
    [](auto, bool confirmed) {
        if (confirmed) {
            log::info("User confirmed!");
        }
    }
);
```

#### Custom Complex Popup:
```cpp
class MyPopup : public Popup<int, std::string> {
protected:
    bool setup(int value, std::string text) override {
        // This is called after the base popup is initialized
        
        auto label = CCLabelBMFont::create(text.c_str(), "bigFont.fnt");
        m_mainLayer->addChild(label);
        
        // Add buttons, inputs, etc.
        
        return true;
    }
    
public:
    static MyPopup* create(int value, std::string text) {
        auto ret = new MyPopup();
        if (ret->initAnchored(300.f, 200.f, value, text)) {
            ret->autorelease();
            return ret;
        }
        delete ret;
        return nullptr;
    }
};

// Use it:
MyPopup::create(42, "Hello")->show();
```

### 9. Web Requests

```cpp
#include <Geode/utils/web.hpp>
#include <Geode/loader/Event.hpp>

class $modify(MenuLayer) {
    struct Fields {
        EventListener<WebTask> m_listener;
    };
    
    bool init() {
        if (!MenuLayer::init()) return false;
        
        // Create request
        web::WebRequest req = web::WebRequest();
        req.header("User-Agent", "MyMod/1.0");
        
        // Send request
        auto task = req.get("https://api.example.com/data");
        
        // Listen for response
        m_fields->m_listener.bind([this](web::WebTask::Event* e) {
            if (web::WebResponse* res = e->getValue()) {
                if (res->ok()) {
                    auto data = res->string();
                    log::info("Response: {}", data);
                    
                    // Parse JSON if needed
                    // auto json = res->json();
                }
            }
        });
        m_fields->m_listener.setFilter(task);
        
        return true;
    }
};
```

### 10. Sprites and Resources

#### Using GD's Built-in Sprites:
```cpp
// From spritesheet (most common)
auto sprite = CCSprite::createWithSpriteFrameName("GJ_playBtn_001.png");

// From individual file
auto sprite = CCSprite::create("mySprite.png");

// IMPORTANT: Always check for null!
if (sprite) {
    sprite->setPosition(100, 100);
    this->addChild(sprite);
}
```

**Finding Sprite Names:**
- Use the Geode VS Code Extension for autocomplete
- Browse GD's spritesheet files
- Use DevTools mod to inspect existing sprites

### 11. The onModify Function

For advanced hook control (priority, accessing hook objects):

```cpp
class $modify(MenuLayer) {
    static void onModify(auto& self) {
        // Set hook priority
        self.setHookPriority("MenuLayer::init", Priority::Early);
        
        // Get hook object
        auto hook = self.getHook("MenuLayer::init");
        if (hook) {
            log::info("Hooked: {}", hook.unwrap()->getDisplayName());
        }
    }
    
    bool init() {
        if (!MenuLayer::init()) return false;
        return true;
    }
};
```

### 12. Common Patterns and Best Practices

#### Pattern: Getting Nodes by ID
```cpp
// Get a node with a known ID
auto menu = this->getChildByID("main-menu");
if (menu) {
    // Do something with menu
}

// Query selector for nested nodes
auto button = this->querySelector("menu-layer > main-menu > play-button");
```

#### Pattern: Using the Prelude Namespace
```cpp
using namespace geode::prelude;

// Now you can use:
// - CCNode, CCSprite, etc. without cocos2d:: prefix
// - All Geode utilities
```

#### Pattern: Logging
```cpp
#include <Geode/utils/cocos.hpp>

log::debug("Debug message");
log::info("Info message");
log::warn("Warning message");
log::error("Error message");

// Format strings
log::info("Value: {}, Name: {}", value, name);
```

#### Pattern: $execute Block
```cpp
// Code that runs when the mod loads (before main menu)
$execute {
    log::info("Mod loaded!");
    
    // Set up event listeners
    // Register custom types
    // etc.
}
```

### 13. Working with Cocos2d Nodes

#### Creating and Positioning Nodes:
```cpp
// Get screen size
auto winSize = CCDirector::get()->getWinSize();

// Create node
auto label = CCLabelBMFont::create("Text", "bigFont.fnt");

// Position (0, 0) is bottom-left
label->setPosition(winSize.width / 2, winSize.height / 2);  // Center
label->setPosition(ccp(100, 100));  // Specific position

// Add as child
this->addChild(label);

// Set Z-order (higher = on top)
this->addChild(label, 10);

// Set scale
label->setScale(1.5f);

// Set opacity
label->setOpacity(128);  // 0-255

// Set color
label->setColor(ccc3(255, 0, 0));  // Red
```

#### Content Size and Anchor Points:
```cpp
// Get size
auto size = node->getContentSize();

// Set size (for some nodes)
node->setContentSize(ccp(200, 100));

// Anchor point (0,0 = bottom-left, 1,1 = top-right, 0.5,0.5 = center)
node->setAnchorPoint(ccp(0.5f, 0.5f));  // Default for most nodes

// Position at anchor point
node->setPosition(100, 100);  // The anchor point will be at (100, 100)
```

### 14. Memory Management

Cocos2d uses reference counting. Key rules:

- **create()** functions return autoreleased objects - don't delete them manually
- **new** requires manual memory management
- Objects added as children are retained automatically
- Use `retain()` to keep objects alive, `release()` when done

```cpp
// Good - autoreleased
auto sprite = CCSprite::create("sprite.png");
this->addChild(sprite);  // Retained by parent

// If you need to keep a pointer:
class $modify(MenuLayer) {
    struct Fields {
        CCSprite* m_mySprite = nullptr;
    };
    
    bool init() {
        if (!MenuLayer::init()) return false;
        
        m_fields->m_mySprite = CCSprite::create("sprite.png");
        m_fields->m_mySprite->retain();  // Keep it alive
        this->addChild(m_fields->m_mySprite);
        
        return true;
    }
    
    ~MenuLayer() {
        if (m_fields->m_mySprite) {
            m_fields->m_mySprite->release();  // Clean up
        }
    }
};
```

### 15. Common Gotchas and Solutions

#### Gotcha: Forgetting to Call Original
```cpp
// BAD - will break the game!
bool init() {
    auto label = CCLabelBMFont::create("Hi", "bigFont.fnt");
    this->addChild(label);
    return true;
}

// GOOD
bool init() {
    if (!MenuLayer::init()) return false;  // Call original!
    auto label = CCLabelBMFont::create("Hi", "bigFont.fnt");
    this->addChild(label);
    return true;
}
```

#### Gotcha: Wrong Sprite Creation Function
```cpp
// If sprite is in spritesheet, use:
auto sprite = CCSprite::createWithSpriteFrameName("GJ_button_01.png");

// If sprite is standalone file, use:
auto sprite = CCSprite::create("myfile.png");

// Using wrong function returns nullptr!
```

#### Gotcha: Not Checking for nullptr
```cpp
// BAD - will crash if sprite doesn't exist
auto sprite = CCSprite::createWithSpriteFrameName("nonexistent.png");
sprite->setPosition(100, 100);  // CRASH!

// GOOD
auto sprite = CCSprite::createWithSpriteFrameName("nonexistent.png");
if (sprite) {
    sprite->setPosition(100, 100);
    this->addChild(sprite);
} else {
    log::error("Failed to create sprite");
}
```

### 16. Including Headers

Always include the necessary headers:

```cpp
// For modifying a class
#include <Geode/modify/ClassName.hpp>

// For Geode utilities
#include <Geode/Geode.hpp>

// Use the prelude (recommended)
#include <Geode/DefaultInclude.hpp>
using namespace geode::prelude;

// For specific features
#include <Geode/utils/web.hpp>      // Web requests
#include <Geode/loader/Event.hpp>   // Events
#include <Geode/ui/GeodeUI.hpp>     // Geode UI events
```

## Quick Reference: Common Tasks

**Add a button to MenuLayer:**
```cpp
#include <Geode/modify/MenuLayer.hpp>
using namespace geode::prelude;

class $modify(MyMod, MenuLayer) {
    bool init() {
        if (!MenuLayer::init()) return false;
        
        auto menu = this->getChildByID("bottom-menu");
        auto btn = CCMenuItemSpriteExtra::create(
            ButtonSprite::create("My Button"),
            this,
            menu_selector(MyMod::onMyButton)
        );
        menu->addChild(btn);
        menu->updateLayout();
        
        return true;
    }
    
    void onMyButton(CCObject*) {
        FLAlertLayer::create("Title", "Message", "OK")->show();
    }
};
```

**Track player jumps:**
```cpp
class $modify(PlayerObject) {
    struct Fields {
        int m_jumpCount = 0;
    };
    
    void pushButton(PlayerButton button) {
        m_fields->m_jumpCount++;
        log::info("Jumps: {}", m_fields->m_jumpCount);
        PlayerObject::pushButton(button);
    }
};
```

**Add a setting:**
```json
// In mod.json
{
  "settings": {
    "my-toggle": {
      "type": "bool",
      "name": "Enable Feature",
      "description": "Enables the cool feature",
      "default": true
    }
  }
}
```

```cpp
// In code
if (Mod::get()->getSettingValue<bool>("my-toggle")) {
    // Feature is enabled
}
```

## Testing Changes

geode build 

## Summary

When helping users with Geode SDK:
1. **Always start with `$modify`** for hooking
2. **Always call the original** in init functions
3. **Use Fields** for adding member variables
4. **Check for nullptr** when creating sprites
5. **Use layouts** instead of manual positioning when possible
6. **Follow Cocos2d patterns** - create with create(), add with addChild()
7. **Use the prelude namespace** for cleaner code
8. **Leverage events** for mod communication
9. **Reference the official docs** at docs.geode-sdk.org for deeper details

The Geode SDK makes GD modding significantly easier by abstracting away the complexity of hooking, providing utilities, and enforcing compatibility between mods.
