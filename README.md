# paloalto

<p><strong>Demo of basic use cases of Palo Alto Firewall automation with Ansible</strong></p>
<ol>
  <li>Automating FW operational tasks<ul>
      <li>Displaying fact information from firewall</li>
      <li>Executing operational mode  commands on firewall</li>
      <li>Creating administrator accounts  </li>    
  </ul></li>  
  <br>
  <li>Automating FW configuration<ul>
      <li>Configuring networking elements (interfaces, virtual router, static routes)</li>
      <li>Configuring zones and objects</li>
      <li>Configuring security policies</li>
  </ul></li>  
  <br>
  <li>Example Ansible automation workflows<ul>
      <li>Automated FW provisioning</li>
        <img src="https://github.com/mzdyb/paloalto/assets/49950423/e9c6c673-df8c-451b-be9f-0e28a438ddbd" alt="Automated FW provisioning" style="display: block; margin-left: auto; margin-right: auto;">
      <br><br>
      <li>Automated FW deprovisioning</li> 
        <img src="https://github.com/mzdyb/paloalto/assets/49950423/33f1f006-1340-4409-b01b-5b99ac8b10c2" alt="Automated FW deprovisioning" style="display: block; margin-left: auto; margin-right: auto;">    
  </ul></li>
  <br>
  Provision or deprovision action is decided based on the values of the variables fw_config_state and fw_rules_state collected via AAP surveys

</ol>
<br>
<>More examples can be found in Palo Alto Networks github Ansible repository https://github.com/PaloAltoNetworks/ansible-playbooks<br>
