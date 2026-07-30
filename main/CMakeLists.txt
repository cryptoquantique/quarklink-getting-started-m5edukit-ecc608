idf_component_register(SRCS "main.c"
                    INCLUDE_DIRS "." "../include")

# Import the library, specifying a target name and the library path.
# The private dependencies of the library is also specified.
add_prebuilt_library(quarklink_client "../lib/libquarklink-client-m5edukit-ecc608-v2.0.2-info.a"
                    PRIV_REQUIRES nvs_flash esp_http_client esp_https_ota app_update mbedtls esp-cryptoauthlib)

# `main` calls a function from the library, so link it to `main`
target_link_libraries(${COMPONENT_LIB} PRIVATE quarklink_client)

# Select the ATECC608 key provider expected by the prebuilt library headers.
target_compile_definitions(${COMPONENT_LIB} PRIVATE QL_CFG_KEYPROV=QL_KEYPROV_ESPIDF_ATECC608)
