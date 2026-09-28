include $(TOPDIR)/rules.mk

PKG_NAME:=fastfetch
PKG_VERSION:=2.66.0
PKG_RELEASE:=1

PKG_SOURCE:=$(PKG_NAME)-$(PKG_VERSION).tar.gz
PKG_SOURCE_URL:=https://codeload.github.com/altman08/fastfetch/tar.gz/refs/tags/v$(PKG_VERSION)?
PKG_HASH:=c175b0e4fc8f3f23affc0fe43516fbd0c31f03c32d9ea2dd31190f657c1da358

PKG_LICENSE:=MIT
PKG_LICENSE_FILES:=LICENSE
PKG_BUILD_PARALLEL:=1

include $(INCLUDE_DIR)/package.mk
include $(INCLUDE_DIR)/cmake.mk

CMAKE_OPTIONS += \
	-DCMAKE_BUILD_TYPE=MinSizeRel \
	-DBUILD_FLASHFETCH=OFF \
	-DSET_TWEAK=OFF \
	-DINSTALL_LICENSE=OFF \
	-DENABLE_WAYLAND=OFF \
	-DENABLE_XCB_RANDR=OFF \
	-DENABLE_XRANDR=OFF \
	-DENABLE_GIO=OFF \
	-DENABLE_DCONF=OFF \
	-DENABLE_DBUS=OFF \
	-DENABLE_SQLITE3=OFF \
	-DENABLE_RPM=OFF \
	-DENABLE_ELF=OFF \
	-DENABLE_LUA=OFF \
	-DENABLE_QUICKJS=OFF \
	-DENABLE_LIBZFS=OFF

define Package/fastfetch
  SECTION:=utils
  CATEGORY:=Utilities
  TITLE:=Fast system information tool
  URL:=https://github.com/fastfetch-cli/fastfetch
endef

define Package/fastfetch/description
 Fastfetch is a neofetch-like tool for fetching system information and displaying it in a pretty way.
endef

define Package/fastfetch/install
	$(INSTALL_DIR) $(1)/usr/bin
	$(INSTALL_BIN) $(PKG_BUILD_DIR)/fastfetch $(1)/usr/bin/fastfetch
endef

$(eval $(call BuildPackage,fastfetch))
