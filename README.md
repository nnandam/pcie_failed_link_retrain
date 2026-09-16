# pcie_failed_link_retrain
Linux Kernel Subsystem Engineering and PCIe Fabric Telemetry Code Archives.
The Constraint & Panics
During our platform bring-up phase for a dense x86 dual-host architecture routing subsystem, we encountered an edge case where specific modular PCIe switch boards would consistently fail to recover full link speed after a heavy hardware flap or Surprise Link Down event. They remained trapped at Gen1 (2.5GT/s) speeds instead of elevating to Gen3 (8.0GT/s), severely bottlenecking system cluster performance.  

🛠️ The Architectural Solution
Identified a bug in the function pcie_failed_link_retarin() in the kernel tree (drivers/pci/quirks.c)
Pointer to vanilla Linux kernel code quirks.c https://elixir.bootlin.com/linux/v6.6/source/drivers/pci/quirks.c

```c
bool pcie_failed_link_retrain(struct pci_dev *dev) {
	static const struct pci_device_id ids[] = {
		{ PCI_VDEVICE(ASMEDIA, 0x2824) }, /* ASMedia ASM2824 */
		{}
	};
	u16 lnksta, lnkctl2;

	if (!pci_is_pcie(dev) || !pcie_downstream_port(dev) ||
	    !pcie_cap_has_lnkctl2(dev) || !dev->link_active_reporting)
		return false;

	pcie_capability_read_word(dev, PCI_EXP_LNKCTL2, &lnkctl2);
	pcie_capability_read_word(dev, PCI_EXP_LNKSTA, &lnksta);
	if ((lnksta & (PCI_EXP_LNKSTA_LBMS | PCI_EXP_LNKSTA_DLLLA)) ==
	    PCI_EXP_LNKSTA_LBMS) {
		pci_info(dev, "broken device, retraining non-functional downstream link at 2.5GT/s\n");

		lnkctl2 &= ~PCI_EXP_LNKCTL2_TLS;
		lnkctl2 |= PCI_EXP_LNKCTL2_TLS_2_5GT;
		pcie_capability_write_word(dev, PCI_EXP_LNKCTL2, lnkctl2);

		if (pcie_retrain_link(dev, false)) {
			pci_info(dev, "retraining failed\n");
			return false;
		}

		pcie_capability_read_word(dev, PCI_EXP_LNKSTA, &lnksta);
	}

	if ((lnksta & PCI_EXP_LNKSTA_DLLLA) &&
	    (lnkctl2 & PCI_EXP_LNKCTL2_TLS) == PCI_EXP_LNKCTL2_TLS_2_5GT &&
	    pci_match_id(ids, dev)) {
		u32 lnkcap;

		pci_info(dev, "removing 2.5GT/s downstream link speed restriction\n");
		pcie_capability_read_dword(dev, PCI_EXP_LNKCAP, &lnkcap);
		lnkctl2 &= ~PCI_EXP_LNKCTL2_TLS;
		lnkctl2 |= lnkcap & PCI_EXP_LNKCAP_SLS;
		pcie_capability_write_word(dev, PCI_EXP_LNKCTL2, lnkctl2);

		if (pcie_retrain_link(dev, false)) {
			pci_info(dev, "retraining failed\n");
			return false;
		}
	}
```
	return true;
}
