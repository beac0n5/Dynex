add_definitions(-DSTATICLIB -DMINIUPNP_STATICLIB)

file(GLOB_RECURSE BlockchainExplorer BlockchainExplorer/*)
file(GLOB_RECURSE Common Common/*)
file(GLOB_RECURSE ConnectivityTool ConnectivityTool/*)
file(GLOB_RECURSE Crypto crypto/*)
file(GLOB_RECURSE DynexCNCore DynexCNCore/* DynexCNConfig.h)
file(GLOB_RECURSE DynexCNProtocol DynexCNProtocol/*)
file(GLOB_RECURSE Daemon Daemon/*)
file(GLOB_RECURSE Dynexchip Dynexchip/*)
file(GLOB_RECURSE GreenWallet GreenWallet/*)
file(GLOB_RECURSE Http HTTP/*)
file(GLOB_RECURSE InProcessNode InProcessNode/*)
file(GLOB_RECURSE Logging Logging/*)
file(GLOB_RECURSE NodeRpcProxy NodeRpcProxy/*)
file(GLOB_RECURSE P2p P2p/*)
file(GLOB_RECURSE Mnemonics Mnemonics/*)
file(GLOB_RECURSE Rpc Rpc/*)
file(GLOB_RECURSE Serialization Serialization/*)
file(GLOB_RECURSE SimpleWallet SimpleWallet/*)
if(MSVC)
file(GLOB_RECURSE System System/* Platform/Windows/System/*)
elseif(APPLE)
file(GLOB_RECURSE System System/* Platform/OSX/System/* Platform/Posix/System/*)
else()
file(GLOB_RECURSE System System/* Platform/Linux/System/* Platform/Posix/System/*)
endif()
file(GLOB_RECURSE Transfers Transfers/*)
file(GLOB_RECURSE Wallet Wallet/*)
file(GLOB_RECURSE WalletLegacy WalletLegacy/*)
file(GLOB_RECURSE JsonRpcServer JsonRpcServer/*)
file(GLOB_RECURSE PaymentGate PaymentGate/*)
file(GLOB_RECURSE PaymentGateService PaymentGateService/*)

source_group("" FILES $${Common} ${ConnectivityTool} ${Crypto} ${DynexCNCore} ${DynexCNProtocol} ${Daemon} ${JsonRpcServer} ${Http} ${Logging} ${NodeRpcProxy} ${P2p} ${Rpc} ${Serialization} ${Mnemonics} ${SimpleWallet} ${System} ${Transfers} ${Wallet} ${WalletLegacy})

add_library(BlockchainExplorer ${BlockchainExplorer})
add_library(Common ${Common})
add_library(Crypto ${Crypto})
add_library(DynexCNCore ${DynexCNCore})
add_library(Http ${Http})
add_library(InProcessNode ${InProcessNode})
add_library(Logging ${Logging})
add_library(NodeRpcProxy ${NodeRpcProxy})
add_library(Rpc ${Rpc})
add_library(P2P ${DynexCNProtocol} ${P2p})
add_library(Mnemonics ${Mnemonics})
add_library(Serialization ${Serialization})
add_library(System ${System})
add_library(Transfers ${Transfers})
add_library(Wallet ${Wallet} ${WalletLegacy})
add_library(PaymentGate ${PaymentGate})
add_library(JsonRpcServer ${JsonRpcServer})

add_executable(ConnectivityTool ${ConnectivityTool})
add_executable(Daemon ${Daemon})
add_executable(SimpleWallet ${SimpleWallet})
add_executable(PaymentGateService ${PaymentGateService})
add_executable(GreenWallet ${GreenWallet})

target_link_libraries(ConnectivityTool DynexCNCore Logging Crypto P2P Rpc Http Serialization Common System ${Boost_LIBRARIES} ${CURL_LIBRARIES})
target_link_libraries(Daemon DynexCNCore P2P Rpc Serialization System Http Logging Common Crypto BlockchainExplorer libminiupnpc-static ${Boost_LIBRARIES} ${CURL_LIBRARIES})
target_link_libraries(SimpleWallet Mnemonics Wallet NodeRpcProxy Transfers Rpc Http Serialization DynexCNCore System Logging Common Crypto ${Boost_LIBRARIES} ${CURL_LIBRARIES})
target_link_libraries(PaymentGateService PaymentGate JsonRpcServer Wallet NodeRpcProxy Transfers DynexCNCore Crypto P2P Rpc Http Serialization System Logging Common InProcessNode BlockchainExplorer libminiupnpc-static ${Boost_LIBRARIES} ${CURL_LIBRARIES})
target_link_libraries(GreenWallet PaymentGate JsonRpcServer Wallet NodeRpcProxy Transfers DynexCNCore Crypto P2P Rpc Http Serialization System Logging Common InProcessNode BlockchainExplorer libminiupnpc-static ${Boost_LIBRARIES} ${CURL_LIBRARIES})

if (MSVC)
  target_link_libraries(System ws2_32)
else()
  target_link_libraries(GreenWallet resolv)
endif()

add_dependencies(Rpc version)
add_dependencies(ConnectivityTool version)
add_dependencies(Daemon version)
add_dependencies(SimpleWallet version)
add_dependencies(PaymentGateService version)
add_dependencies(P2P version)
add_dependencies(GreenWallet version)

set_property(TARGET ConnectivityTool PROPERTY OUTPUT_NAME "connectivity_tool")
set_property(TARGET SimpleWallet PROPERTY OUTPUT_NAME "simplewallet")
set_property(TARGET PaymentGateService PROPERTY OUTPUT_NAME "walletd")
set_property(TARGET Daemon PROPERTY OUTPUT_NAME "dynexd")
set_property(TARGET GreenWallet PROPERTY OUTPUT_NAME "greenwallet")

if (DEFINED GUI)
  add_subdirectory(WalletGui)
endif()
