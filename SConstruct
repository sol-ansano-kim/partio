import os
import sys
import glob
import excons
import excons.tools.gl as gl
import excons.tools.glut as glut
import excons.tools.zlib as zlib
import excons.tools.python as python


use_zlib = (excons.GetArgument("use-zlib", 1, int) != 0)
excons.SetArgument("use-zlib", 1 if use_zlib else 0)

use_seexpr = (excons.GetArgument("use-seexpr", 0, int) != 0)
excons.SetArgument("use-c++11", 1 if use_seexpr else 0)


env = excons.MakeBaseEnv()

libdefs = []

cmndefs = []
cmncppflags = ""
cmnincdirs = []
cmnlibdirs = []
cmnlibs = []
cmncusts = []

if sys.platform == "win32":
   cmndefs.extend(["PARTIO_WIN32", "_CRT_SECURE_NO_WARNINGS", "_CRT_NONSTDC_NO_DEPRECATE"])
   if excons.warnl != "all":
     cmncppflags += " -wd4267 -wd4244"
else:
   cmncppflags += " -Wno-unused-parameter"

if use_zlib:
   cmncusts.append(zlib.Require)
   libdefs.append("PARTIO_USE_ZLIB")

swig_opts = ""

if use_seexpr:
   cmndefs.append("PARTIO_USE_SEEXPR")
   swig_opts = "-DPARTIO_USE_SEEXPR"
   seexpr_inc, seexpr_lib = excons.GetDirs("seexpr")
   if seexpr_inc:
      cmnincdirs.append(seexpr_inc)
   if seexpr_lib:
      cmnlibdirs.append(seexpr_lib)
   cmnlibs.append("SeExpr2")
   if sys.platform == "win32":
      cmndefs.append("SEEXPR_WIN32")

SConscript("gto/SConstruct")
Import("RequireGto")
cmncusts.append(RequireGto(static=True))

partio_headers = env.Install(excons.OutputBaseDirectory() + "/include", glob.glob("src/lib/*.h"))

py_prefix = python.ModulePrefix() + "/" + python.Version()

swig_builder = Builder(action='$SWIG -o $TARGET -c++ -python -Wall %s $SOURCE' % swig_opts, suffix='.cpp', src_suffix='.i')
env.Append(BUILDERS={"SwigGen": swig_builder})

prjs = [
   {"name": "partio",
    "type": "staticlib",
    "defs": cmndefs + libdefs,
    "cppflags": cmncppflags,
    "incdirs": cmnincdirs,
    "srcs": glob.glob("src/lib/core/*.cpp") +
            glob.glob("src/lib/io/*.cpp") +
            glob.glob("src/lib/*.cpp"),
    "custom": cmncusts
   },
   {"name": "partattr",
    "type": "program",
    "alias": "tools",
    "defs": cmndefs,
    "incdirs": cmnincdirs,
    "cppflags": cmncppflags,
    "incdirs": cmnincdirs,
    "srcs": ["src/tools/partattr.cpp"],
    "libdirs": cmnlibdirs,
    "staticlibs": ["partio"] + cmnlibs,
    "custom": cmncusts
   },
   {"name": "partinfo",
    "type": "program",
    "alias": "tools",
    "defs": cmndefs,
    "cppflags": cmncppflags,
    "incdirs": cmnincdirs,
    "srcs": ["src/tools/partinfo.cpp"],
    "libdirs": cmnlibdirs,
    "staticlibs": ["partio"] + cmnlibs,
    "custom": cmncusts
   },
   {"name": "partconv",
    "type": "program",
    "alias": "tools",
    "defs": cmndefs,
    "cppflags": cmncppflags,
    "incdirs": cmnincdirs,
    "srcs": ["src/tools/partconv.cpp"],
    "libdirs": cmnlibdirs,
    "staticlibs": ["partio"] + cmnlibs,
    "custom": cmncusts
   },
   {"name": "partview",
    "type": "program",
    "alias": "tools",
    "defs": cmndefs,
    "cppflags": cmncppflags + ("" if sys.platform != "darwin" else " -Wno-deprecated-declarations"),
    "incdirs": cmnincdirs,
    "srcs": ["src/tools/partview.cpp"],
    "libdirs": cmnlibdirs,
    "staticlibs": ["partio"] + cmnlibs,
    "custom": cmncusts + [glut.Require, gl.Require]
   }
]

# Python
swig = excons.GetArgument("with-swig", None)
if not swig:
   swig = excons.Which("swig")
if swig:
   env["SWIG"] = swig
   gen = env.SwigGen(["src/py/partio_wrap.cpp", "src/py/partio.py"], "src/py/partio.i")
   prjs.append({"name": "_partio",
                "type": "dynamicmodule",
                "alias": "python",
                "ext": python.ModuleExtension(),
                "prefix": py_prefix,
                "defs": cmndefs,
                "cppflags": cmncppflags,
                "incdirs": cmnincdirs,
                "srcs": [gen[0]],
                "libdirs": cmnlibdirs,
                "staticlibs": ["partio"] + cmnlibs,
                "custom": cmncusts + [python.SoftRequire],
                "install": {py_prefix: [gen[1]]}})
# Tests
tests = filter(lambda x: x.endswith("_main.cpp"), glob.glob("src/tests/*.cpp"))
for test in tests:
   name = os.path.basename(test).replace("_main.cpp", "")
   srcs = glob.glob(test.replace("_main.cpp", "*.cpp"))
   prjs.append({"name": name,
                "type": "program",
                "alias": "tests",
                "prefix": "../tests",
                "defs": cmndefs,
                "cppflags": cmncppflags,
                "incdirs": cmnincdirs + ["src/lib"],
                "srcs": srcs,
                "libdirs": cmnlibdirs,
                "staticlibs": ["partio"] + cmnlibs,
                "custom": cmncusts})
for test in ["makecircle", "makeline", "testcluster", "testse"]:
   prjs.append({"name": test,
                "type": "program",
                "alias": "tests",
                "prefix": "../tests",
                "defs": cmndefs,
                "cppflags": cmncppflags,
                "incdirs": cmnincdirs + ["src/lib"],
                "srcs": ["src/tests/%s.cpp" % test],
                "libdirs": cmnlibdirs, 
                "staticlibs": ["partio"] + cmnlibs,
                "custom": cmncusts})

build_opts = """PARTIO OPTIONS
  use-zlib=0|1   : Enable zlib compression.                 [1]
  use-seexpr=0|1 : Enable particle expression using SeExpr. [0]"""
excons.AddHelpOptions(partio=build_opts)

tgts = excons.DeclareTargets(env, prjs)

env.Depends(tgts["partio"], partio_headers)