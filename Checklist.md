# Firmware Modification Checklist

Use this checklist to guide you through the firmware creation.

---

## **1. Gather Donor Device Information**

Collect the following information from the donor PCIe device:

- [ ] **Device ID** (`0xXXXX`)
- [ ] **Vendor ID** (`0xYYYY`)
- [ ] **Subsystem ID** (`0xZZZZ`)
- [ ] **Subsystem Vendor ID** (`0xWWWW`)
- [ ] **Revision ID** (`0xRR`)
- [ ] **Class Code** (`0xCCCCCC`)
- [ ] **Base Address Registers (BARs) Configuration**:
  - [ ] **BAR0** to **BAR5** sizes, types (Memory or I/O), prefetchable status
- [ ] **Capabilities**:
  - [ ] Power Management settings
  - [ ] MSI/MSI-X settings
- [ ] **Device Serial Number (DSN)** (`0xXXXXXXXXYYYYYYYY`)

---

## **2. Modify Firmware Configuration Files**

### **2.1. Open Firmware Configuration File**

### **2.2. Update Device and Vendor IDs**

- [ ] **Modify** `cfg_deviceid`:

  ```verilog
  cfg_deviceid <= 16'hXXXX;  // Replace XXXX with donor's Device ID
  ```

- [ ] **Modify** `cfg_vendorid`:

  ```verilog
  cfg_vendorid <= 16'hYYYY;  // Replace YYYY with donor's Vendor ID
  ```

### **2.3. Update Subsystem IDs and Revision ID**

- [ ] **Modify** `cfg_subsysid`:

  ```verilog
  cfg_subsysid <= 16'hZZZZ;  // Replace ZZZZ with donor's Subsystem ID
  ```

- [ ] **Modify** `cfg_subsysvendorid`:

  ```verilog
  cfg_subsysvendorid <= 16'hWWWW;  // Replace WWWW with donor's Subsystem Vendor ID
  ```

- [ ] **Modify** `cfg_revisionid`:

  ```verilog
  cfg_revisionid <= 8'hRR;  // Replace RR with donor's Revision ID
  ```

### **2.4. Update Class Code**

- [ ] **Modify** `cfg_classcode`:

  ```verilog
  cfg_classcode <= 24'hCCCCCC;  // Replace CCCCCC with donor's Class Code
  ```

### **2.5. Insert Device Serial Number (DSN)**

- [ ] **Modify** `cfg_dsn`:

  ```verilog
  cfg_dsn <= 64'hXXXXXXXXYYYYYYYY;  // Replace with donor's DSN
  ```

---

## **3. Customize Vivado Project**

### **3.1. Generate Vivado Project Files**

- [ ] **Open Vivado** and run the appropriate Tcl script in the Tcl Console:

  ```tcl
  cd <project_directory>
  source vivado_generate_project_<your_board>.tcl -notrace
  ```

  - Replace `<project_directory>` with your project path.
  - Replace `<your_board>` with your FPGA board identifier (e.g., `squirrel`).

### **3.2. Open Generated Project**

- [ ] **File**: Open the generated `.xpr` project file in Vivado.

---

## **4. Modify PCIe IP Core**

### **4.1. Open PCIe IP Core Configuration**

- [ ] **File to Edit**: `pcie_7x_0.xci`
- [ ] **Action**: Right-click and select **Customize IP**.

### **4.2. Set Device Identifiers**

- [ ] **Device ID**: Set to donor's Device ID.
- [ ] **Vendor ID**: Set to donor's Vendor ID.
- [ ] **Subsystem ID**: Set to donor's Subsystem ID.
- [ ] **Subsystem Vendor ID**: Set to donor's Subsystem Vendor ID.
- [ ] **Revision ID**: Set to donor's Revision ID.
- [ ] **Class Code**: Set to donor's Class Code.

### **4.3. Configure BARs**

For each BAR (BAR0 to BAR5):

- [ ] **Enable/Disable**: Match donor device.
- [ ] **Type**: Memory (32-bit or 64-bit) or I/O.
- [ ] **Size**: Set to donor's BAR size.
- [ ] **Prefetchable**: Match donor's setting.

### **4.4. Match PCIe Link Parameters**

- [ ] **Maximum Link Speed**: Set to match donor device (e.g., **Gen2**, **Gen3**).
- [ ] **Link Width**: Set to match donor device (e.g., **x1**, **x4**).

### **4.5. Configure Capabilities**

- [ ] **Enable Power Management** if the donor device supports it.
- [ ] **Enable MSI/MSI-X** if the donor device supports it.

### **4.6. Apply Changes and Lock IP Core**

- [ ] **Apply**: Click **OK** to save IP core settings.
- [ ] **Lock IP Core**: In the Tcl Console, run:

  ```tcl
  set_property -name {IP_LOCKED} -value true -objects [get_ips pcie_7x_0]
  ```

---

## **5. Adjust Firmware for Advanced Features**

### **5.1. Update Capability Pointer**

- [ ] **File to Edit**: `pcileech_pcie_cfg_a7.sv`
- [ ] **Modify** `cfg_cap_pointer`:

  ```verilog
  cfg_cap_pointer <= 8'hXX;  // Replace XX with donor's capability pointer
  ```

### **5.2. Adjust Max Payload and Read Request Sizes**

- [ ] **In PCIe IP Core**: Set **Max Payload Size** and **Max Read Request Size** to match donor device.

- [ ] **In Firmware (`pcileech_pcie_cfg_a7.sv`)**:

  ```verilog
  max_payload_size_supported <= 3'bYYY;  // Binary value corresponding to donor's Max Payload Size
  ```

  - **Mapping**:

    | Payload Size (Bytes) | Binary Value |
    |----------------------|--------------|
    | 128                  | `3'b000`     |
    | 256                  | `3'b001`     |
    | 512                  | `3'b010`     |
    | 1024                 | `3'b011`     |
    | 2048                 | `3'b100`     |
    | 4096                 | `3'b101`     |

### **5.3. Implement Power Management Logic**

- [ ] **File to Edit**: `pcileech_pcie_cfg_a7.sv`
- [ ] **Add Logic** to handle power state transitions if applicable.

### **5.4. Implement MSI/MSI-X Interrupts**

- [ ] **In PCIe IP Core**: Enable MSI or MSI-X capability and set the number of supported vectors.

- [ ] **In Firmware (`pcileech_pcie_tlp_a7.sv`)**:

  - [ ] **Add Interrupt Logic** to handle MSI/MSI-X interrupts.

---

## **6. Adjust BAR Handling in Firmware**

### **6.1. Open BAR Controller File**

- [ ] **File to Edit**: `pcileech_tlps128_bar_controller.sv`
- [ ] **Location**: `pcileech-fpga/pcileech-wifi-main/src/pcileech_tlps128_bar_controller.sv`

### **6.2. Update BAR Address Decoding**

- [ ] **Modify** the address decoding logic to match the BAR sizes and addresses of the donor device.

### **6.3. Implement BAR Access Logic**

For each enabled BAR:

- [ ] **Implement Read/Write Handlers** corresponding to the BAR's purpose.

---

## **7. Implement TLP Handling**

### **7.1. Open TLP Handling File**

- [ ] **File to Edit**: `pcileech_pcie_tlp_a7.sv`

### **7.2. Modify TLP Processing Logic**

- [ ] **Implement** logic to handle specific TLP types required by the donor device:

  - [ ] Memory Read Requests
  - [ ] Memory Write Requests
  - [ ] Configuration Read/Write Requests
  - [ ] Vendor-Defined Messages

### **7.3. Ensure TLP Compliance**

- [ ] **Verify** that TLPs are correctly formatted according to PCIe specifications.

---

## **8. Build and Flash Firmware**

### **8.1. Run Synthesis and Implementation**

In Vivado:

- [ ] **Run Synthesis**
- [ ] **Run Implementation**
- [ ] **Generate Bitstream**

### **8.2. Program the FPGA**

- [ ] **Connect** your FPGA device via JTAG.
- [ ] **Open Hardware Manager**.
- [ ] **Program Device** with the generated bitstream.

> **Note (2024–2026):** Newer boards (e.g., CaptainDMA 35T/75T/100T/M.2) flash via the **WCH CH347 / CH347_OpenOCD** tool over an integrated update port — no separate JTAG dongle required. For automated config-space capture → `.coe` conversion, see the companion repo's `tools/config_space_to_coe.py`.

---

## **9. Test and Validate**

### **9.1. Verify Device Enumeration**

- [ ] **Check** that the FPGA appears as the donor device on the host system.

### **9.2. Install Necessary Drivers**

- [ ] **Use** donor device drivers if required.

### **9.3. Perform Functional Testing**

- [ ] **Test** all functionalities expected from the donor device.

### **9.4. Monitor for Errors**

- [ ] **Check** system logs for any errors or warnings related to the device.

---

## **10. Debug and Optimize as Needed**

### **10.1. Use Integrated Logic Analyzer (ILA)**

- [ ] **Insert** ILA cores to monitor internal signals if necessary.

### **10.2. Analyze PCIe Traffic**

- [ ] **Use** PCIe protocol analyzers to debug communication issues.

### **10.3. Refine Firmware**

- [ ] **Iterate** on your firmware code to fix bugs and improve performance.
