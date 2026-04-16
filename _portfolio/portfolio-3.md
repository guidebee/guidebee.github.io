---
title: "iRobot - OpenAI Gym for Android Games"
excerpt: "C++ reimplementation of scrcpy as an OpenAI Gym environment for training RL agents on Android games<br/><img src='https://github.com/guidebee/irobot/raw/master/docs/irobot_android_new_arch.png'>"
collection: portfolio
---

## iRobot - OpenAI Gym Environment for Android Games

iRobot is an innovative OpenAI Gym environment that enables reinforcement learning agents to interact with and learn from any Android game or application. Built upon the foundation of scrcpy (screen copy), this project represents a complete architectural reimagining and modernization of Android screen mirroring technology for machine learning applications.

### Project Genesis

After successfully developing the Australia ASX Gym Environment for stock market simulation, the next logical challenge was to create a Gym environment for Android games. When discovering scrcpy's impressive performance capabilities—achieving 60fps without requiring device rooting—it became clear this was the ideal foundation for an Android-based RL environment.

### Technical Transformation

**From C to Modern C++**
- Complete port from C to modern C++
- Migration from Meson to CMake build system
- Integration of vcpkg for robust C++ package management
- Improved code maintainability and extensibility

**Architectural Redesign**
- Re-architected to function as a proper OpenAI Gym environment
- Optimized for RL agent integration
- Modular design for easy extension and customization
- Performance-oriented implementation maintaining scrcpy's efficiency

### Key Features

**High-Performance Screen Mirroring**
- 60fps capability for real-time agent interaction
- Low-latency screen capture and input injection
- No root access required on Android devices
- Hardware acceleration support

**OpenAI Gym Integration**
- Standard Gym API for seamless agent training
- Observation space: real-time screen capture (RGB arrays)
- Action space: touch events, key presses, and gestures
- Reward signal customization based on game state

**Modern Development Stack**
- C++ for performance-critical operations
- CMake for cross-platform builds
- vcpkg for dependency management
- Well-documented codebase for contributors

### Use Cases

- Training RL agents to play mobile games
- Automated mobile app testing
- Game AI research and development
- Mobile UI/UX automation
- Performance benchmarking of RL algorithms on real applications

### Technical Advantages

The transition from C to C++ with modern tooling provides:
- Better code organization and maintainability
- Easier integration with ML frameworks
- Simplified dependency management via vcpkg
- More flexible architecture for custom environments
- Enhanced type safety and debugging capabilities

### Resources

**GitHub Repository**
- [iRobot on GitHub](https://github.com/guidebee/irobot)

### Applications

iRobot opens up exciting possibilities for:
- Developing game-playing AI agents without game source code access
- Creating automated testing frameworks for Android applications
- Researching transfer learning across different mobile games
- Building generalized agents that can adapt to various Android interfaces
- Educational platform for teaching RL in interactive, visual environments

This project demonstrates the power of combining established open-source tools with modern software engineering practices to create new research and development platforms. By bridging the gap between Android applications and reinforcement learning frameworks, iRobot enables a new class of mobile AI agents.
