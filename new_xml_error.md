# XML Location Context Implementation

This document describes the implementation of XML location context for error messages in preCICE (Issue #751).

## Overview

The implementation adds file location context (line numbers, column numbers, and surrounding lines) to XML configuration error messages to help users quickly identify and fix configuration errors.

## Changes Made

### 1. ConfigParser Enhancements

#### ConfigParser.hpp
- Added `TagLocation` struct to store line and column information
- Added `m_Location` member to `CTag` struct to track tag positions
- Added `m_parserContext` to store the parser context during parsing
- Added `m_fileContent` to store file content for error reporting
- Added `formatLocationContext()` method to format location context strings

#### ConfigParser.cpp
- Modified `readXmlFile()` to:
  - Store file content in `m_fileContent`
  - Store parser context in `m_parserContext` during parsing
  - Clear parser context after parsing
- Modified `OnStartElement()` to capture line and column information using `xmlSAX2GetLineNumber()` and `xmlSAX2GetColumnNumber()`
- Implemented `formatLocationContext()` to generate formatted multi-line context with line numbers
- Modified `connectTags()` to:
  - Pass location information from `CTag` to `XMLTag`
  - Add location context to error messages for unknown tags and duplicate tags

### 2. XMLTag Enhancements

#### XMLTag.hpp
- Added `_location` member to store tag location
- Added `_fileContent` pointer for accessing file content
- Added `getLocationContext()` method to retrieve formatted location context

#### XMLTag.cpp
- Implemented `getLocationContext()` method
- Modified error messages in `readAttributes()` to include location context:
  - Unknown attribute errors
  - Attribute hint errors
- Modified error messages in `areAllSubtagsConfigured()` to include location context for missing required tags

## Error Message Format

The new error messages now include location context in the following format:

```
ERROR: The tag <test> in the configuration contains an unknown attribute "wrong-attr". Expected attributes are: attribute.
   4 |     <test attribute="value-one" />
   5 >     <test wrong-attribute="value-two" />
         ^
   6 |     <test attribute="value-three" />
```

Where:
- Line numbers are shown with `|` for context lines and `>` for the error line
- The `^` marker shows the column position (when available)
- Context lines before and after the error line are shown (configurable, default is 1)

## Example Error Messages

### Unknown Tag Error
```
ERROR: The configuration contains an unknown tag <solver-interface>. Did you mean <precice-configuration>?
   1 > <solver-interface dimensions="3">
       ^
```

### Unknown Attribute Error
```
ERROR: The tag <participant> in the configuration contains an unknown attribute "naem". Did you mean "name"?
  10 > <participant naem="Fluid">
       ^
```

### Missing Required Tag Error
```
ERROR: Tag <data> was not found but is required to occur at least once.
  15 > <mesh name="FluidMesh">
       ^
  16 |   <use-data name="Forces"/>
```

### Duplicate Tag Error
```
ERROR: Tag <time-window-size> is not allowed to occur multiple times.
  40 > <time-window-size value="0.1" />
       ^
```

## Technical Details

### Location Tracking
- Line and column numbers are captured during SAX parsing using libxml2's `xmlSAX2GetLineNumber()` and `xmlSAX2GetColumnNumber()` functions
- Location information flows from `CTag` (created during parsing) to `XMLTag` (used during validation)
- The parser context is only stored during the `xmlParseChunk()` call and cleared immediately after

### Memory Management
- File content is stored once in `ConfigParser` and referenced (not copied) in `XMLTag` instances
- Parser context is a temporary pointer, valid only during parsing
- Location information is copied from `CTag` to `XMLTag`

### Performance Considerations
- File content is stored for error reporting but only used when errors occur
- Location context formatting is only performed when an error is raised
- Minimal overhead during successful parsing

## Testing

To test the implementation:
1. Create an XML configuration file with intentional errors
2. Run preCICE with the configuration
3. Verify that error messages include line numbers and context

Example test file created: `src/xml/tests/test_location_context.xml`

## Future Enhancements

Potential improvements mentioned in the original issue:
- Show column markers for multi-character attributes/tags
- Handle multi-line XML tags
- Add more vertical context (multiple lines before/after)
- Improve column accuracy for complex tag structures

## Related Issue

GitHub Issue #751: Context for XML error messages
 
