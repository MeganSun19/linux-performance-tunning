
## Ubuntu 24.04 + XRDP 闪退问题总结

在 Ubuntu 24.04 Desktop 中，XRDP与GNOME桌面存在已知兼容性问题，即便关闭 Wayland 仍会导致 XRDP 登录后立即闪退。日志显示
- Window manager exited quickly
- xrdp_mcs_incoming failed
- .xorgxrdp.log 不生成

根因是 XRDP 无法可靠启动 GNOME 的 Xorg 会话。

解决办法：为 XRDP 强制使用 XFCE 桌面环境，并完全绕过 GNOME。
步骤：

## 禁用Wayland:
    vi /etc/gdm3/custome.conf
    [daemon]
    WaylandEnable=false
    重启：
    sudo systemctl restart gdm3


## 安装并指定 XRDP 使用 XFCE

    sudo apt install xfce4 xfce4-goodies
    用以下内容完全覆盖 /etc/xrdp/startwm.sh：

    #!/bin/sh
    export LANG=en_US.UTF-8
    export DESKTOP_SESSION=xfce
    export XDG_SESSION_DESKTOP=xfce
    export XDG_CURRENT_DESKTOP=XFCE
    unset DBUS_SESSION_BUS_ADDRESS
    unset WAYLAND_DISPLAY
    unset XDG_RUNTIME_DIR
    if [ -r "$HOME/.xsession" ]; then
        rm -f "$HOME/.xsession"
    fi
    exec startxfce4


## 清理用户 session 文件：

    rm -f ~/.Xauthority ~/.xsession ~/.xsession-errors
    rm -rf ~/.cache/sessions/*


## 重启 XRDP：

    sudo systemctl restart xrdp xrdp-sesman
    sudo reboot

完成后，XRDP 可稳定登录 XFCE，不再闪退。