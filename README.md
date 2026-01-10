# Digital Circuit Design Course - 2026
## Complete Course Package

Welcome to the Digital Circuit Design course! This folder contains all materials for a comprehensive 15-week course on digital logic and FPGA design.

The website can be found at [here](https://lbaitemple.github.io/ECE2612_DCD/).

---

## 📚 Course Structure

This course consists of 12 modules plus exams and a final project:

| Module | Title | Duration | Key Topics |
|--------|-------|----------|------------|
| **0** | Course Introduction & Software Setup | Week 1 | Quartus, DE10-Lite, First FPGA project |
| **1** | Digital Logic Fundamentals | Weeks 1-2 | Number systems, Boolean algebra, gates |
| **2** | Combinational Logic Design | Weeks 2-3 | K-maps, MUX, decoders, adders |
| **3** | Verilog HDL Basics | Week 4 | Syntax, testbenches, simulation |
| **4** | Number Systems & Binary Representation | Week 5 | Signed numbers, 2's complement, BCD |
| **5** | Binary Arithmetic & ALU Design | Week 6 | Adders, multipliers, ALU |
| **6** | Latches and Flip-Flops | Weeks 7-8 | SR, D, JK, T flip-flops, timing |
| **7** | Sequential Circuit Design | Weeks 8-9 | State machines, analysis & synthesis |
| **8** | Counters, Shift Registers, Timers | Week 10 | Counters, universal shift register |
| **9** | Finite State Machines (FSM) | Weeks 11-12 | Moore/Mealy, datapath+control |
| **10** | Memory Systems | Week 13 | RAM, ROM, FIFO, cache |
| **11** | Advanced Topics | Week 14 | CDC, pipelining, optimization |

---

## 📁 Files in This Folder

```
course/
├── SYLLABUS.md                         # Complete course syllabus
├── README.md                           # This file
├── EXAM_TEMPLATES.md                   # Exam formats and practice problems
│
├── Module_00_Introduction.md           # Module 0: Getting started
├── Module_01_Logic_Fundamentals.md     # Module 1: Digital logic basics
├── Module_02_Combinational_Logic.md    # Module 2: Combinational circuits
├── Module_03_Verilog_HDL.md           # Module 3: Verilog programming
├── Module_04_Number_Systems.md         # Module 4: Number representations
├── Module_05_Binary_Arithmetic.md      # Module 5: Arithmetic circuits
├── Module_06_Latches_Flipflops.md     # Module 6: Storage elements
├── Module_07_Sequential_Design.md      # Module 7: Sequential circuits
├── Module_08_Counters_Registers.md     # Module 8: Counters & registers
├── Module_09_Finite_State_Machines.md  # Module 9: FSM design
├── Module_10_Memory_Systems.md         # Module 10: Memory design
└── Module_11_Advanced_Topics.md        # Module 11: Advanced concepts
```

---

## 🎯 Learning Path

### Beginner (Weeks 1-5)
Start with fundamentals:
1. **Module 0**: Set up your tools
2. **Module 1**: Learn Boolean algebra and gates
3. **Module 2**: Master combinational building blocks
4. **Module 3**: Start coding in Verilog
5. **Module 4-5**: Understand number systems and arithmetic

**Milestone**: Design a complete ALU

### Intermediate (Weeks 6-10)
Build sequential circuits:
6. **Module 6**: Storage with flip-flops
7. **Module 7**: Design state machines
8. **Module 8**: Create counters and registers

**Milestone**: Implement a complex counter system

### Advanced (Weeks 11-14)
Master professional techniques:
9. **Module 9**: FSM-based control systems
10. **Module 10**: Memory subsystems
11. **Module 11**: Industry best practices

**Milestone**: Complete final project

---

## 🛠️ Required Tools

### Software (All Free)
- **Intel Quartus Prime Lite** (23.1std or later)
  - Download: https://www.intel.com/fpga
  - Size: ~6GB
  - Includes ModelSim simulator

- **Drawing Tool** (choose one):
  - Draw.io: https://app.diagrams.net/
  - Lucidchart: https://www.lucidchart.com/

- **Text Editor**:
  - VS Code (recommended): https://code.visualstudio.com/
  - Includes Verilog extensions

### Hardware
- **FPGA Board**: DE10-Lite (Terasic)
  - Purchase: ~$85-100
  - Alternative: Simulation-only option available

### Optional
- Textbook: "Digital Design and Computer Architecture" by Harris & Harris
- Breadboard and logic chips for hands-on experiments

---

## 📊 Grading Breakdown

| Component | Weight | Details |
|-----------|--------|---------|
| Module Assignments (11) | 35% | Weekly assignments, drop lowest |
| Midterm Exam 1 | 15% | Modules 0-3 |
| Midterm Exam 2 | 15% | Modules 4-7 |
| Final Exam | 25% | Comprehensive |
| Final Project | 10% | System design project |

---

## 🚀 Getting Started

1. **Read the Syllabus** (`SYLLABUS.md`)
   - Understand course policies
   - Note important dates
   - Review grading criteria

2. **Start Module 0** (`Module_00_Introduction.md`)
   - Install Quartus Prime
   - Set up your FPGA board
   - Complete Assignment 0

3. **Join Online Resources**
   - Canvas course page
   - Discussion forums
   - Office hours schedule

---

## 📖 How to Use These Materials

### For Each Module:

1. **Read the Module File**
   - Review learning objectives
   - Study topics and examples
   - Watch recommended videos

2. **Practice**
   - Work through examples
   - Try lab exercises
   - Experiment on FPGA

3. **Complete Assignment**
   - Follow requirements carefully
   - Test thoroughly
   - Submit on time

4. **Seek Help When Needed**
   - Office hours
   - Discussion board
   - Study groups

---

## 💡 Tips for Success

### Study Strategies
- **Practice Regularly**: Digital design requires hands-on experience
- **Build Incrementally**: Test small pieces before combining
- **Draw Diagrams**: Visualize circuits and state machines
- **Comment Code**: Future you will thank present you
- **Debug Systematically**: Use simulation, not just FPGA testing

### Common Pitfalls to Avoid
- ❌ Waiting until last minute for assignments
- ❌ Skipping simulation and going straight to hardware
- ❌ Not backing up your work
- ❌ Copying code without understanding
- ❌ Ignoring timing issues

### Best Practices
- ✅ Start assignments early
- ✅ Test thoroughly with testbenches
- ✅ Use version control (Git)
- ✅ Follow Verilog coding standards
- ✅ Document your designs

---

## 🔧 Troubleshooting

### Software Issues
- **Quartus won't compile**: Check for syntax errors, save all files
- **Simulation doesn't work**: Verify testbench, check `timescale`
- **Programming fails**: Check USB connection, device selection

### Hardware Issues
- **FPGA not detected**: Check drivers, try different USB port
- **Circuit doesn't work**: Verify pin assignments, check syntax
- **Unexpected behavior**: Review timing constraints, add synchronizers

### Getting Help
1. Check module documentation
2. Search online (Stack Overflow, FPGA forums)
3. Ask on course discussion board
4. Attend office hours
5. Email instructor for private issues

---

## 📚 Additional Resources

### Online Learning
- **HDLBits**: Interactive Verilog exercises
- **FPGA4Student**: Tutorials and projects
- **Nandland**: Verilog basics and FPGA tutorials
- **YouTube Channels**: Neso Academy, Ben Eater

### Reference Materials
- Intel Quartus Documentation
- Verilog IEEE Standard (IEEE 1364)
- SystemVerilog LRM
- FPGA Prototyping guides

### Communities
- reddit.com/r/FPGA
- forum.digikey.com
- Intel FPGA forums
- EDABoard forums

---

## 🎓 Final Project Ideas

Choose a project that interests you and demonstrates your skills:

### Beginner Projects
- Digital clock with alarm
- Calculator with 7-segment display
- Simon Says game
- Traffic light controller with sensors

### Intermediate Projects
- VGA pattern generator
- Serial communication interface (UART/SPI)
- Music synthesizer
- Stopwatch with lap timer

### Advanced Projects
- Simple 8-bit processor
- Video game (Pong, Snake, Tetris)
- Digital oscilloscope
- PWM motor controller with feedback

---

## 📅 Important Reminders

- **Back up your work**: Use Cloud storage or Git
- **Test early and often**: Don't wait until the deadline
- **Ask questions**: No question is too simple
- **Form study groups**: Learn from peers
- **Have fun**: Digital design is creative and rewarding!

---

## 📞 Contact & Support

**Instructor**: [Name]  
**Email**: [Email]  
**Office**: [Location]  
**Office Hours**: [Times]  

**Teaching Assistants**:
- Lab support available during scheduled times
- Canvas messaging for questions

**Technical Support**:
- IT Help Desk for Canvas/computer issues
- Quartus support via Intel forums

---

## 🌟 Course Outcomes

Upon successful completion of this course, you will be able to:

✅ Design and analyze digital circuits using Boolean algebra  
✅ Implement combinational and sequential logic in Verilog  
✅ Program and test designs on FPGA hardware  
✅ Design finite state machines for control systems  
✅ Create memory structures and interfaces  
✅ Apply professional design practices and optimizations  
✅ Debug and verify digital systems systematically  
✅ Complete an independent digital system design project  

---

## 🚀 Let's Build Something Amazing!

Digital circuit design is at the heart of modern technology - from smartphones to spacecraft. This course will give you the skills to design, implement, and test real digital systems.

**Ready to get started? Open `SYLLABUS.md` and `Module_00_Introduction.md`!**

---

*Version 1.0 - January 2026*  
*Last Updated: December 29, 2025*
