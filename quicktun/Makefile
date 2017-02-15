#
# Copyright (C) 2017 OpenWrt.org
#
# This is free software, licensed under the GNU General Public License v2.
# See /LICENSE for more information.
#

include $(TOPDIR)/rules.mk

PKG_NAME:=quicktun
PKG_VERSION:=2.2.5
PKG_RELEASE:=4

PKG_SOURCE_PROTO:=git
PKG_SOURCE_URL:=https://github.com/UCIS/QuickTun.git
PKG_SOURCE_SUBDIR:=$(PKG_NAME)-$(PKG_VERSION)
PKG_SOURCE_VERSION:=2d0c6a9cda8c21f921a5d1197aeee92e9568ca39
PKG_SOURCE:=$(PKG_NAME)-$(PKG_VERSION)-$(PKG_SOURCE_VERSION).tar.gz

PKG_BUILD_DEPENDS:=nacl

include $(INCLUDE_DIR)/package.mk

define Package/quicktun
  SECTION:=net
  CATEGORY:=Network
  DEPENDS:=+kmod-tun
  TITLE:=QuickTun is a simple and secure VPN software
  URL:=http://wiki.ucis.nl/QuickTun
  SUBMENU:=VPN
endef

define Package/quicktun/description
 QuickTun is a simple and secure VPN software
endef

define Package/quicktun/conffiles
/etc/config/quicktun
endef

define Build/Compile
	( \
		cd $(PKG_BUILD_DIR); \
		mkdir -p obj out; \
		export CPATH=$(STAGING_DIR)/usr/include/nacl; \
		\
		$(TARGET_CC) $(TARGET_CFLAGS) $(TARGET_CPPFLAGS) -c -DCOMBINED_BINARY  -DQT_VERSION=\"$(PKG_VERSION)\"       src/proto.raw.c         -o obj/proto.raw.o; \
		$(TARGET_CC) $(TARGET_CFLAGS) $(TARGET_CPPFLAGS) -c -DCOMBINED_BINARY  -DQT_VERSION=\"$(PKG_VERSION)\"      src/proto.nacl0.c       -o obj/proto.nacl0.o; \
		$(TARGET_CC) $(TARGET_CFLAGS) $(TARGET_CPPFLAGS) -c -DCOMBINED_BINARY  -DQT_VERSION=\"$(PKG_VERSION)\"      src/proto.nacltai.c     -o obj/proto.nacltai.o; \
		$(TARGET_CC) $(TARGET_CFLAGS) $(TARGET_CPPFLAGS) -c -DCOMBINED_BINARY  -DQT_VERSION=\"$(PKG_VERSION)\"      src/proto.salty.c     -o obj/proto.salty.o; \
		$(TARGET_CC) $(TARGET_CFLAGS) $(TARGET_CPPFLAGS) -c -DCOMBINED_BINARY  -DQT_VERSION=\"$(PKG_VERSION)\"      src/run.combined.c      -o obj/run.combined.o; \
		$(TARGET_CC) $(TARGET_CFLAGS) $(TARGET_CPPFLAGS) -c  -DQT_VERSION=\"$(PKG_VERSION)\"                        src/common.c            -o obj/common.o; \
		$(TARGET_CC) $(TARGET_CFLAGS) $(TARGET_CPPFLAGS) -c  -DQT_VERSION=\"$(PKG_VERSION)\"                        src/randombytes.c            -o obj/randombytes.o; \
		$(TARGET_CC)  $(TARGET_LDFLAGS) -o out/quicktun.combined obj/common.o obj/run.combined.o obj/proto.raw.o obj/proto.nacl0.o obj/proto.nacltai.o obj/proto.salty.o obj/randombytes.o \
			-lnacl; \
		\
		$(TARGET_CC) $(TARGET_CFLAGS) $(TARGET_CPPFLAGS) $(TARGET_LDFLAGS) -o out/quicktun.keypair -DQT_VERSION=\"$(PKG_VERSION)\"   src/keypair.c src/randombytes.c        -lnacl \
	)
endef

define Package/quicktun/install
	$(INSTALL_DIR) $(1)/usr/sbin
	$(INSTALL_BIN) $(PKG_BUILD_DIR)/out/quicktun.{combined,keypair} $(1)/usr/sbin/
	$(LN) quicktun.combined $(1)/usr/sbin/quicktun

	$(INSTALL_DIR) $(1)/etc/init.d/
	$(INSTALL_BIN) files/$(PKG_NAME).init $(1)/etc/init.d/$(PKG_NAME)
	$(INSTALL_DIR) $(1)/etc/config
	$(INSTALL_CONF) files/$(PKG_NAME).config $(1)/etc/config/$(PKG_NAME)
endef

$(eval $(call BuildPackage,quicktun))
