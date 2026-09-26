# -*- mode: ruby -*-
# زیرساخت آزمایشگاه Todo - پروژه‌ی ۱
# این فایل فقط «سخت‌افزار» را تعریف می‌کند: ماشین، رم، CPU، شبکه.
# نصب و کانفیگ نرم‌افزار کار Ansible است (مرحله‌ی ۷).

NETWORK = "192.168.60"
BOX     = "bento/ubuntu-24.04"

# جدول سرورها: یک جا تعریف می‌شود، همه‌جا استفاده می‌شود.
SERVERS = {
  "lb"   => { ip: "#{NETWORK}.10", memory: 512,  cpus: 1 },
  "app1" => { ip: "#{NETWORK}.21", memory: 768,  cpus: 1 },
  "app2" => { ip: "#{NETWORK}.22", memory: 768,  cpus: 1 },
  "db"   => { ip: "#{NETWORK}.30", memory: 1024, cpus: 1 },
}

# خطوط /etc/hosts از روی همان جدول ساخته می‌شود.
# برچسب «# todo-lab» بعداً برای پیدا کردن و پاک کردن همین خطوط به کار می‌آید.
HOSTS_ENTRIES = SERVERS.map { |name, s| "#{s[:ip]}  #{name}  # todo-lab" }.join("\n")

Vagrant.configure("2") do |config|
  config.vm.box = BOX
  config.vm.synced_folder ".", "/vagrant", disabled: true

  SERVERS.each do |name, s|
    config.vm.define name do |node|
      node.vm.hostname = name

      node.vm.network "private_network",
        ip: s[:ip],
        libvirt__network_name: "todo-net",
        libvirt__dhcp_enabled: false,
        libvirt__forward_mode: "none"

      node.vm.provider "libvirt" do |lv|
        lv.memory = s[:memory]
        lv.cpus   = s[:cpus]
      end

      node.vm.provision "shell", name: "hosts", inline: <<~SHELL
        sed -i '/# todo-lab$/d' /etc/hosts
        echo '#{HOSTS_ENTRIES}' >> /etc/hosts
      SHELL
    end
  end
end
