include (LibAddMacros)

add_plugin (mozprefs
	SOURCES
		mozprefs.h
		mozprefs.c
	LINK_ELEKTRA
		elektra-utility
	ADD_TEST
	INSTALL_TEST_DATA
	)
