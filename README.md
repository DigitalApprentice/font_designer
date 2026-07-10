# LED Font Designer

The **LED Font Designer** (`font_editor.html`) is an interactive, browser-based graphical utility for creating, editing, and exporting custom bitmap fonts. It is designed to format fonts specifically for the `CLeds` LED display system, outputting either MicroPython-compatible Python modules or C-style header files.

---

## Getting Started

1. Navigate to the `tools` directory in your file explorer.
2. Open `font_editor.html` in any modern web browser (Chrome, Edge, or Firefox recommended for File System API support).
3. The editor runs completely locally and offline. No server is required.

---

## Interface Overview

The interface is divided into three functional columns:

1. **Left Sidebar (Configuration & Character List)**:
   - **Dimensions**: Configure the cell width and height (e.g., `5`x`7`, `8`x`8`, `8`x`16`).
   - **ASCII Range**: Define the start and end ASCII character boundaries (e.g., standard printable characters start at `32` (Space) and end at `126` (`~`)).
   - **Character Grid**: Clickable buttons for each character in the defined range. Select a character to edit its bitmap representation in the main canvas.

2. **Center Section (Main Editor Canvas)**:
   - **Grid Canvas**: An interactive pixel matrix where clicking or dragging toggles pixels between on (active color) and off (dark background).
   - **Canvas Utilities**:
     - **Clear**: Turn off all pixels in the active character.
     - **Invert**: Swap on and off states for all pixels.
     - **Shift Buttons**: Shift the drawing up, down, left, or right (pixels wrapping or padding appropriately).
     - **Undo / Redo**: Quick reversion options (supports standard `Ctrl+Z` hotkeys).

3. **Right Sidebar (Data & Exporters)**:
   - **Import Text Area**: Paste custom text/hex data representing a font array, then click **Import** to load it back into the designer.
   - **Export Configuration**: Choose the output file name and target export format (`.py` or `.h`).
   - **Save File / Save As**: Utilizes the modern web File System Access API to write changes directly back to your workspace files, falling back to standard browser downloads if permissions are not granted.
   - **Code Output Preview**: Displays a real-time generated code representation of the current font.

---

## Font Data Structures

The exporter packages the character matrix data as column-major arrays (each integer represents one vertical column of pixels, with the least significant bit (LSB) at the top):

### 1. Python Format (.py)
The Python exporter generates a single dictionary containing metadata and a flat list of column integers.

```python
# Real-time preview format
font_data = {
    "Width": 5,
    "Height": 8,
    "Start": 32,
    "End": 126,
    "Data": [
        0x00, 0x00, 0x00, 0x00, 0x00, # Space (ASCII 32)
        0x00, 0x00, 0x5f, 0x00, 0x00, # ! (ASCII 33)
        # ... sequential bytes for all characters
    ]
}
```

### 2. C Header Format (.h)
The C exporter defines constants and a raw byte array containing the sequential column bitmaps.

```c
#ifndef FONT_5X8_H
#define FONT_5X8_H

#include <stdint.h>

#define FONT_WIDTH  5
#define FONT_HEIGHT 8

static const uint8_t font_5x8_data[] = {
    0x00, 0x00, 0x00, 0x00, 0x00, // Space (ASCII 32)
    0x00, 0x00, 0x5F, 0x00, 0x00, // ! (ASCII 33)
    // ...
};

#endif // FONT_5X8_H
```

---

## Integration Guide

Below are integration patterns for importing the exported fonts into your applications.

### 1. MicroPython Integration (with `leddisplay.LEDDisplaySystem`)

Save your exported file (e.g. as `myfont_5x8.py`) in your board's flash memory next to your application script.

```python
import leddisplay
import myfont_5x8

# 1. Initialize your LED Display System
# (Adjust pins, dimensions, and layout based on your configuration)
display = leddisplay.LEDDisplaySystem(rows=138, cols=4, render_direction=0, spacing=2)

# 2. Load the external font into the display system
# Specify a registration name and pass the data dictionary
display.load_external_font("custom_5x8", myfont_5x8.font_data)

# 3. Set it as the active rendering font
display.set_font("custom_5x8")

# 4. Write text to the display
display.draw_string(x=0, y=0, text="HELLO WORLD")
display.show()
```

### 2. Direct Python Lookup (Raw Usage)

If you are writing a custom text renderer that does not use the `leddisplay` module:

```python
import myfont_5x8

def get_char_columns(char):
    char_code = ord(char)
    fd = myfont_5x8.font_data
    start = fd["Start"]
    end = fd["End"]
    width = fd["Width"]

    if start <= char_code <= end:
        index = char_code - start
        # Retrieve the slice of columns for this specific character
        return fd["Data"][index * width : (index + 1) * width]
    else:
        # Fallback to empty/space columns
        return [0] * width

# Usage:
cols_for_a = get_char_columns('A')
print("Column bitmasks for 'A':", [hex(col) for col in cols_for_a])
```

### 3. Custom C Module Integration (ESP32 / MicroPython usermod)

To embed the custom font directly inside the custom C user module (`custom_modules/leddisplay/leddisplay.c`):

1. Save the exported header (e.g., `font_5x8.h`) in `custom_modules/leddisplay/`.
2. Include the header at the top of your `leddisplay.c` file:
   ```c
   #include "font_5x8.h"
   ```
3. Initialize the display font configuration referencing the compiled array:
   ```c
   // Inside initialization method
   self->font_width = FONT_WIDTH;
   self->font_height = FONT_HEIGHT;
   self->font_spacing = 1; // Default spacing

   size_t data_len = sizeof(font_5x8_data);
   
   // Allocate and load font values
   uint16_t *allocated_data = m_new(uint16_t, data_len);
   for (size_t i = 0; i < data_len; i++) {
       allocated_data[i] = font_5x8_data[i];
   }

   // Assign to the custom font lookup slots
   self->font_data = allocated_data;
   strcpy(self->fonts[0].name, "custom_5x8");
   self->fonts[0].width = FONT_WIDTH;
   self->fonts[0].height = FONT_HEIGHT;
   self->fonts[0].data = allocated_data;
   ```

### 4. Direct C Lookup (Raw C Code)

For generic C/C++ microcontrollers (e.g. Arduino or ESP-IDF):

```c
#include <stdio.h>
#include "font_5x8.h"

// Returns pointer to the columns of the given ASCII character (or NULL if out of bounds)
const uint8_t* get_char_columns(char ch) {
    int start_char = 32; // The ASCII index the font starts at
    int total_chars = sizeof(font_5x8_data) / FONT_WIDTH;
    
    int index = (int)ch - start_char;
    if (index < 0 || index >= total_chars) {
        return NULL; // Character not supported by font range
    }
    
    // Return pointer to the start of the character's column data
    return &font_5x8_data[index * FONT_WIDTH];
}

int main() {
    const uint8_t *cols = get_char_columns('A');
    if (cols != NULL) {
        printf("Columns for 'A':\n");
        for (int i = 0; i < FONT_WIDTH; i++) {
            printf("  Col %d: 0x%02X\n", i, cols[i]);
        }
    }
    return 0;
}
```
