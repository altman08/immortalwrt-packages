include $(TOPDIR)/rules.mk

PKG_NAME:=xray-core
PKG_VERSION:=26.10.29
PKG_RELEASE:=1

PKG_SOURCE:=$(PKG_NAME)-$(PKG_VERSION).tar.gz
PKG_SOURCE_URL:=https://codeload.github.com/XTLS/Xray-core/tar.gz/v$(PKG_VERSION)?
PKG_HASH:=ee22c6f21855f70a52e19bff02fa71f4b28ce071d83ff9154fafcaa0f315a642

PKG_MAINTAINER:=Tianling Shen <cnsztl@immortalwrt.org>
PKG_LICENSE:=MPL-2.0 BSD-3-Clause
PKG_LICENSE_FILES:=LICENSE

PKG_BUILD_DIR:=$(BUILD_DIR)/Xray-core-$(PKG_VERSION)
PKG_BUILD_DEPENDS:=golang/host
PKG_BUILD_PARALLEL:=1
PKG_BUILD_FLAGS:=no-mips16

GO_PKG:=github.com/xtls/xray-core
GO_PKG_BUILD_PKG:=github.com/xtls/xray-core/main
GO_PKG_LDFLAGS_X:= \
	$(GO_PKG)/core.build=OpenWrt \
	$(GO_PKG)/core.version=$(PKG_VERSION)

include $(INCLUDE_DIR)/package.mk
include ../../lang/golang/golang-package.mk

# Keep the overlay private to Xray; runtime DT detection gates AN7581 use.
# Generate it with the same versioned Go environment as the package build.
ifeq ($(GO_ARCH),arm64)
  define Build/Compile
	$(GO_PKG_VARS) python3 $(PKG_BUILD_DIR)/tools/eip93/overlay.py --go go --out $(PKG_BUILD_DIR)/.eip93-overlay
	$(call GoPackage/Build/Compile,-overlay=$(PKG_BUILD_DIR)/.eip93-overlay/overlay.json)
  endef
endif

define Package/xray/template
  TITLE:=A proxy platform to bypass network restrictions
  SECTION:=net
  CATEGORY:=Network
  URL:=https://xtls.github.io
endef

define Package/xray-core
  $(call Package/xray/template)
  DEPENDS:=$(GO_ARCH_DEPENDS) +ca-bundle \
	+TARGET_airoha_an7581:kmod-crypto-hw-eip93 \
	+TARGET_airoha_an7581:kmod-crypto-cbc \
	+TARGET_airoha_an7581:kmod-crypto-gcm \
	+TARGET_airoha_an7581:kmod-crypto-user
endef

define Package/xray-example
  $(call Package/xray/template)
  TITLE+= (example configs)
  DEPENDS:=xray-core
  PKGARCH:=all
endef

define Package/xray/description
  Xray, Penetrates Everything. It helps you to build your own computer network.
  It secures your network connections and thus protects your privacy.
endef

define Package/xray-core/description
  $(call Package/xray/description)
endef

define Package/xray-example/description
  $(call Package/xray/description)

  This includes example configuration files for xray-core.
endef

define Package/xray-core/conffiles
/etc/xray/
/etc/config/xray
endef

define Package/xray-core/install
	$(call GoPackage/Package/Install/Bin,$(PKG_INSTALL_DIR))
	$(INSTALL_DIR) $(1)/usr/bin/
	$(INSTALL_BIN) $(PKG_INSTALL_DIR)/usr/bin/main $(1)/usr/bin/xray

	$(INSTALL_DIR) $(1)/etc/xray/
	$(INSTALL_DATA) $(CURDIR)/files/config.json.example $(1)/etc/xray/

	$(INSTALL_DIR) $(1)/etc/config/
	$(INSTALL_CONF) $(CURDIR)/files/xray.conf $(1)/etc/config/xray
	$(INSTALL_DIR) $(1)/etc/init.d/
	$(INSTALL_BIN) $(CURDIR)/files/xray.init $(1)/etc/init.d/xray
endef

define Package/xray-example/install
	$(INSTALL_DIR) $(1)/etc/xray/
	$(INSTALL_DATA) $(CURDIR)/files/vpoint_socks_vmess.json $(1)/etc/xray/
	$(INSTALL_DATA) $(CURDIR)/files/vpoint_vmess_freedom.json $(1)/etc/xray/
endef

$(eval $(call BuildPackage,xray-core))
$(eval $(call BuildPackage,xray-example))
