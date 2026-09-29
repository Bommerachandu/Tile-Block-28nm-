# Tile Block – RTL to GDSII Implementation | 28nm

This project is a complete **RTL-to-GDSII physical design implementation of a Tile block at 28nm**.

I worked through the design from synthesis to final physical verification using **Synopsys Design Compiler (DC), Synopsys ICC2, and Tcl scripting**.

The main focus of this project was to understand how a design behaves at each physical design stage and, more importantly, how to debug problems when the expected results are not achieved.

---

## What I Worked On

The complete flow covered:

**RTL → Synthesis → Floorplanning → Power Planning → Placement → CTS → Routing → Timing Closure → Physical Verification → GDSII**

During the implementation, I worked on timing violations, macro placement, congestion, power-grid issues, and post-route timing.

One of the most useful parts of the project was seeing how a decision made during floorplanning could affect congestion and timing much later in the flow.

---

## Tools

* **Synopsys Design Compiler (DC)** – RTL synthesis
* **Synopsys ICC2** – Physical design implementation
* **Tcl** – Scripting and flow automation
* **Linux** – EDA environment

---

# Design Details

| Parameter           |   Value |
| ------------------- | ------: |
| Technology          |    28nm |
| Clock Period        |  1.7 ns |
| Total Cells         | 413,935 |
| Sequential Cells    | 101,188 |
| Combinational Cells | 312,709 |
| Macros              |      18 |
| Initial Utilization |     70% |
| Final Utilization   |   80.7% |

The design is **macro-dominant**, with 18 macros. Because of this, macro placement and the available routing resources had a significant impact on the implementation.

---

# My Approach

I did not treat the flow as simply a sequence of commands.

At every stage, I checked the reports and tried to understand:

* What is causing the current problem?
* Is it a timing issue or a physical issue?
* Can the problem be fixed at the current stage?
* Will the fix create another problem somewhere else?
* What happens to the QoR after making the change?

This helped me understand the relationship between **floorplanning, placement, congestion, CTS, routing, and timing closure**.

---

# 1. Synthesis

I started with synthesis using **Synopsys Design Compiler**.

The RTL was synthesized into a gate-level netlist using the target 28nm technology libraries.

After synthesis, I checked the initial timing and design reports.

### Initial QoR

```text
Setup WNS       : -0.11 ns
TNS             : -81.21 ns
Violations      : 2458
```

The design was not timing clean at this point, which was expected because physical effects had not yet been considered.

---

## An Issue I Found During Synthesis

During the timing analysis, I noticed **unconstrained endpoints**.

Some of the endpoints were related to latch-based logic, so I did not want to simply ignore the warnings and continue with physical implementation.

I investigated the affected cells using reports such as:

```tcl
report_cell
```

The timing setup and endpoints were reviewed before continuing with the implementation.

### What I learned

This made me understand an important point:

> A timing report is only useful when the design is properly constrained.

If endpoints are unconstrained, the reported timing picture may not represent the actual design requirements.

---

# 2. Floorplanning

After synthesis, I moved the design into **ICC2**.

At the floorplanning stage, I worked on:

* Core size
* Core aspect ratio
* Macro placement
* Standard-cell area
* Routing resources
* Congestion
* Power planning requirements

The 18 macros made this stage particularly important.

I initially tried a floorplan where several macros became vertically clustered.

That created a routing bottleneck.

---

# 3. Macro Placement and Congestion

This was one of the main problems I had to work through.

The initial congestion overflow was around:

```text
2.5%
```

Instead of waiting until routing to deal with the problem, I went back and looked at the floorplan.

I checked the relationship between the macro locations, core dimensions, and available routing resources.

I then modified the **core sizing and aspect ratio** and adjusted the floorplan.

After the changes, the congestion overflow was reduced to approximately:

```text
0.2%
```

### Before

```text
Congestion Overflow ≈ 2.5%
```

### After

```text
Congestion Overflow ≈ 0.2%
```

### What I learned

This was one of the biggest lessons from the project:

> A routing problem can sometimes actually be a floorplanning problem.

Moving cells or optimizing routing alone is not always the best solution. If the floorplan is creating a bottleneck, it is better to fix the problem earlier.

---

# 4. Power Planning

After working on the floorplan, I created the power distribution network.

I looked at:

* Power rings
* Power straps
* VDD/VSS connectivity
* Via connections
* Routing resources
* Power-grid DRC

During this stage, I encountered **power-grid DRC issues and missing-via problems**.

I used:

```tcl
check_pg_drc
```

to investigate the power-grid problems.

I then worked on the **via stacking and strap pitch** to improve the connectivity and resolve the reported issues.

After the changes, the power-grid connectivity was brought to a clean state.

---

# 5. Placement

Once the floorplan and power structure were in place, I moved to standard-cell placement.

The main things I monitored were:

* Cell density
* Congestion
* Timing
* Placement legality
* Wirelength
* Routing resources

The placement stage produced the following timing results:

```text
Setup WNS : -0.24 ns
TNS       : -10.67 ns
Violations: 780
```

The design was still not timing clean, so optimization continued.

One thing I noticed during this stage was that improving one metric does not necessarily improve everything else.

For example, moving cells to improve timing can sometimes increase congestion.

That trade-off became an important part of the optimization process.

---

# 6. Clock Tree Synthesis

After placement, I moved to **Clock Tree Synthesis (CTS)**.

The purpose was to build a clock distribution network that could deliver the clock to the sequential elements while controlling:

* Clock skew
* Clock latency
* Transition
* Clock routing
* Timing impact

The CTS stage changed the timing picture again.

### CTS QoR

```text
Setup WNS : -0.32 ns
TNS       : -46.93 ns
Violations: 1416
```

At first glance, the timing looked worse than the placement stage.

However, this was useful because CTS introduces the actual clock-tree effects into the timing analysis.

I used the CTS reports to identify the paths that needed further optimization.

---

# 7. Routing

After CTS and optimization, I moved to routing.

The routing stage connected the standard cells, macros, clock network, and other design elements while considering the physical design rules.

I checked for:

* Routing congestion
* Connectivity
* Shorts
* Opens
* Antenna violations
* DRC
* Timing

After routing, the design was analyzed with the post-route timing information.

This stage was especially important because routing introduces parasitic effects that can change the timing compared with earlier stages.

---

# 8. Timing Closure

Timing closure was one of the main goals of the project.

The design did not start with positive timing.

The reported setup WNS changed during the implementation:

| Stage      |    Setup WNS |    TNS | Violations |
| ---------- | -----------: | -----: | ---------: |
| Synthesis  |     -0.11 ns | -81.21 |       2458 |
| Placement  |     -0.24 ns | -10.67 |        780 |
| CTS        |     -0.32 ns | -46.93 |       1416 |
| Post Route | **+0.02 ns** |  **0** |      **0** |

The final post-route result was:

```text
Setup WNS = +0.02 ns
TNS       = 0
Violations = 0
```

I also checked hold timing.

The final reported hold WNS was:

```text
Hold WNS = +0.28 ns
```

So the final reported timing results were:

```text
Setup WNS : +0.02 ns
Hold WNS  : +0.28 ns
TNS       : 0
Violations: 0
```

---

# 9. Physical Verification

After timing closure, I checked the physical verification results.

The final reported results were:

```text
DRC Violations      : 0
Shorts              : 0
Opens               : 0
Antenna Violations  : 0
```

These checks were important because achieving positive timing alone does not mean that the physical implementation is complete.

The design also needs to satisfy the relevant physical and connectivity checks.

---

# Final Results

The final implementation achieved the following reported results:

| Check              | Final Result |
| ------------------ | -----------: |
| Setup WNS          | **+0.02 ns** |
| Setup TNS          |     **0 ns** |
| Hold WNS           | **+0.28 ns** |
| Timing Violations  |        **0** |
| DRC Violations     |        **0** |
| Shorts             |        **0** |
| Opens              |        **0** |
| Antenna Violations |        **0** |
| Final Utilization  |    **80.7%** |

The final GDSII database was generated after completing the reported implementation and verification steps.

---

# Problems I Solved

## 1. Unconstrained Endpoints

**Problem:**
Unconstrained endpoints were found during synthesis timing analysis.

**What I did:**
Investigated the affected cells and latch-based endpoints using cell-level reports and reviewed the timing setup.

**Result:**
The timing analysis was cleaned up before continuing with the implementation.

---

## 2. Macro Clustering and Congestion

**Problem:**
The initial macro arrangement created a routing bottleneck.

**Initial congestion overflow:** `2.5%`

**What I changed:**

* Reviewed macro distribution
* Modified the core aspect ratio
* Adjusted core sizing
* Rechecked congestion

**Result:**

`2.5% → 0.2%`

This showed me how strongly macro placement can influence the rest of the physical design flow.

---

## 3. Power-Grid DRC

**Problem:**
Power-grid DRC issues and missing-via problems appeared during power planning.

**What I used:**

```tcl
check_pg_drc
```

**What I changed:**

* Via stacking
* Strap pitch
* PG connectivity

**Result:**
The reported PG connectivity issues were resolved and the final physical checks were clean.

---

# Tcl Scripting

I also used **Tcl scripting** during the implementation to make repetitive tasks easier and to help with report generation and flow execution.

The scripting was useful for tasks such as:

* Running ICC2 commands
* Generating reports
* Checking design status
* Repeating implementation steps
* Collecting QoR
* Running verification checks

Some of the ICC2 checks used during the project included:

```tcl
report_cell
check_pg_drc
```

Using Tcl also helped me become more comfortable working in a Linux-based EDA environment.

---

# What I Learned From This Project

The biggest takeaway from this project was that physical design is not just about knowing commands.

A change at one stage can affect several other stages.

For example:

```text
Macro Placement
       ↓
Congestion
       ↓
Placement
       ↓
Routing
       ↓
Parasitics
       ↓
Timing
```

Similarly:

```text
Floorplan
    ↓
Power Planning
    ↓
Placement
    ↓
CTS
    ↓
Routing
    ↓
Timing Closure
    ↓
Physical Verification
```

Working through the problems helped me understand why each stage exists and how the stages are connected.

---

# Repository Structure

```text
Tile_RTL_to_GDSII_28nm/
│
├── README.md
│
├── docs/
│   ├── project_report.pdf
│   ├── floorplan.png
│   ├── Congestion.png
│   ├── Congestion_CTS.png
│   └── celldensity.png
│
├── rtl/
│
├── constraints/
│
├── scripts/
│   ├── synthesis/
│   ├── floorplan/
│   ├── placement/
│   ├── cts/
│   ├── routing/
│   └── signoff/
│
├── reports/
│   ├── synthesis/
│   ├── placement/
│   ├── cts/
│   ├── route/
│   └── signoff/
│
└── outputs/
    └── gds/
```

The actual repository structure can be adjusted depending on which design files and reports are available for sharing.

---

# Final Summary

This project gave me hands-on exposure to a complete **28nm RTL-to-GDSII implementation flow** using Synopsys DC and ICC2.

Rather than only running the flow, I worked through several practical implementation problems, including **unconstrained endpoints, macro-related congestion, power-grid DRC issues, and timing closure**.

The final reported implementation achieved:

```text
Technology       : 28nm
Clock Period     : 1.7 ns
Macros           : 18
Total Cells      : 413,935
Final Utilization: 80.7%

Setup WNS        : +0.02 ns
Hold WNS         : +0.28 ns
TNS              : 0
Timing Violations: 0

DRC              : 0
Shorts           : 0
Opens             : 0
Antenna          : 0
```

The project helped me build a stronger practical understanding of how **floorplanning, placement, CTS, routing, timing, congestion, power planning, and physical verification work together in a real physical design flow**.

---

## Author

**BOMMERA CHANDU**

**Tools:** Synopsys ICC2 | Tcl

**Technology:** 28nm

**Project:** RTL-to-GDSII Physical Design
