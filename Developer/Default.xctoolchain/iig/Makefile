CXX := ${HOST_CXX}
SRCS = iig.cpp
OBJS = iig.o
CXXFLAGS += -std=gnu++17
.if "${.MAKE.OS}" == "Darwin"
# iig is a HOST tool: it must use the macOS SDK's libc++ headers. The ravynOS
# target include dir carries a libc++ with a different ABI inline-namespace
# (nqe210106 vs the system __1), so codecvt<>::id resolves to a symbol the
# system libc++.dylib never exports. Drop the target include for this tool.
CFLAGS := ${CFLAGS:N-I*Default.xctoolchain/include}
CXXFLAGS := ${CXXFLAGS:N-I*Default.xctoolchain/include}
.endif

all: ${TOOLS}/iig

# The build undefines __APPLE__ globally (host-tool flags), which suppresses the
# libc++ locale template instantiations (std::codecvt<>::id) — force libc++ and
# let the Darwin toolchain provide the locale symbols.
.if "${.MAKE.OS}" == "Darwin"
IIG_LDLIBS = -stdlib=libc++ -lc++
.endif

${TOOLS}/iig: ${OBJS}
	${CXX} -o ${.TARGET} ${OBJS} ${IIG_LDLIBS}

