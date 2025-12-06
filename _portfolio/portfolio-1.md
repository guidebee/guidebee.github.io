---
title: "Australia ASX Gym Environment"
excerpt: "OpenAI Gym-based virtual stock exchange platform with 10 years of real ASX market data<br/><img src='https://media.licdn.com/dms/image/sync/v2/C5627AQGj1ST2YUjAGA/articleshare-shrink_480/articleshare-shrink_480/0/1764289126509?e=1765551600&v=beta&t=Rye9RsJS4D0zWUehNqFmKx8c9D3DUrJE9c-MwkBI2Lk'>"
collection: portfolio
---

## Australia ASX Gym Environment

The Australia ASX Gym Environment is a sophisticated virtual stock exchange trading platform built on OpenAI Gym. It provides a realistic simulation environment that allows reinforcement learning agents to buy and sell stocks in real-time market conditions.

### Key Features

**Comprehensive Market Data**
- SQLite database containing 10 years of real transaction data
- Covers approximately 2,000 ASX listed companies
- Spans 11 different market sectors

**Flexible Configuration**
- Simulate the entire ASX market or select specific companies of interest
- Optional brokerage fee simulation
- Configurable start dates for historical backtesting
- Weekly data update tool to maintain current market information

**Multiple Render Modes**
The environment supports three visualization modes:
- **Human Mode**: Renders interactive visualizations on-screen for human observation
- **RGB Array Mode**: Returns RGB image arrays without on-screen rendering, ideal for video creation
- **ANSI Mode**: Displays text-based output on the screen without images

**Data and Visualization**
- Utilizes mpl_finance library for professional stock data rendering
- Automatically saves episode data (daily portfolio values) for further analysis
- Generates and stores images at each simulation step
- Creates video-ready image sequences in human and RGB array modes

### Simulation Output

During each training episode, the ASX Gym Environment generates:
- Step-by-step portfolio value tracking
- Visual representations of market conditions and trading activity
- Episode summary data for performance analysis
- Image sequences suitable for creating training videos

### GitHub Repository

[View on GitHub](https://github.com/guidebee/asx_gym)
