# Configuration Troubleshooting

---

**The VPP agent obtained configuration data from the northbound and the VPP crashed**

This could happen if the configuration data was incomplete or corrupted. The VPP agent does not recognize the data so VPP cannot handle it properly. Check for data correctness and [reference Ligato Support](../intro/faq.md#where-can-i-find-support-for-ligato).

It may also may happen randomly for any configuration if the VPP version is compatible but with a different commit ID than what is listed in the [vpp.env](https://github.com/ligato/vpp-agent/blob/master/vpp.env). You can also [reference Ligato Support](../intro/faq.md#where-can-i-find-support-for-ligato).

---

**The vpp-agent obtained the configuration data from the northbound with a printed error of "No reply received within timeout ..."**
</br>
</br>
Similar problem as mentioned above. VPP received incomplete or incorrect data which caused it to hang resulting in no reply sent back to the vpp-agent. Check the data correctness and [reference Ligato Support](../intro/faq.md#where-can-i-find-support-for-ligato).

---

**The VPP agent obtained the configuration data from the northbound but I do not see the configured item in the VPP**<br>

Two possibilities:

* The given configuration item is dependent on another configuration item which is missing. For example, the FIB entry cannot exist without a defined interface and bridge domain. In such a case, the VPP agent caches the configuration and waits for the dependencies. Reference the [KVS troubleshooting guide](../developer-guide/kvs-troubleshooting.md) to assist in identifying and resolving this problem.

* The plugin expected to handle the configuration is not loaded. For example, in order to configure IPSec, the IPSec plugin must be a part of the application plugins.

---

**The VPP agent obtained the configuration data from the northbound but the plugin procedure ends up with this error "VPPApiError: (message with numeric return value)"**

This error indicates a binary API call failure. It means data was successfully processed by the VPP agent, but VPP refused it for some reason. Check that the data is complete and in the right format. Also, these kinds of errors can suddenly appear while switching to a different VPP version. In that case, it is most likely a bug in the VPP agent caused by an external dependency. [Reference Ligato Support](../intro/faq.md#where-can-i-find-support-for-ligato)

---

**The VPP agent obtained the configuration data from the northbound with a printed error of "No reply received within timeout ..."**
</br>
</br>
Similar problem as the first one mentioned above. VPP received incomplete or incorrect data which caused it to hang resulting in no reply sent back to the VPP agent. Check the data correctness and [reference Ligato Support](../intro/faq.md#where-can-i-find-support-for-ligato).

---

