# SwiftR-Switcheroo-Calling-Compiled-Swift-from-R-Part-1

Hello, Swift!

To keep this post short (since I’ll be adding this entire concept to the SwiftR tome), we’ll be super-focused and just build a shared library we can dynamically load into R. That library will have one function which will be to let us say hello to the planet with a customized greeting.

Make a new directory for this effort (I called mine greetings) and create a greetings.swift file with the following contents:

NOTE: Please substitute a real @ sign for the ＠ used here. One of the extensions that auto-converts @- things to Twitter links is busted and turns them all to Twitter links.

All this code is also in this gist.

＠_cdecl("greetings_from")
public func greetings_from(_ who: SEXP) -> SEXP {
  print("Greetings, 🌎, it's \(String(cString: R_CHAR(STRING_ELT(who, 0))))!")
  return(R_NilValue)
}

Before I explain what’s going on there, also create a geetings.h file with the following contents:

#define USE_RINTERNALS

#include 
#include 

const char* R_CHAR(SEXP x);

In the Swift file, there’s a single function that takes an R SEXP and converts it into a Swift String which is then routed to stdout (not a “great” R idiom, but benign enough for an intro example). Swift functions aren’t C functions and on their own do not adhere to C calling conventions. Unfortunately R’s ability to work with dynamic library code requires such a contract to be in place. Thankfully, the Swift Language Overlords provided us with the ability to instruct the compiler to create library code that will force the calling conventions to be C-like (that’s what the ＠cdecl is for).

We’re using SEXP, some R C-interface functions, and even the C version of NULL in the Swift code, but we haven’t done anything in the Swift file to tell Swift about the existence of these elements. That’s what the C header file is for (I added the R_CHAR declaration since complex C macros don’t work in Swift).

Now, all we need to do is make sure the compiler knows about the header file (which is a “bridge” between C and Swift), where the R framework is, and that we want to generate a library vs a binary executable file as we compile the code. Make sure you’re in the same directory as both the .swift and .h file and execute the following at a terminal prompt:

swiftc \
  -I /Library/Frameworks/R.framework/Headers \ # where the R headers are
  -F/Library/Frameworks \                      # where the R.framework lives
  -framework R \                               # we want to link against the R framework
  -import-objc-header greetings.h \            # this is our bridging header which will make R C things available to Swift
  -emit-library \                              # we want a library, not an exe
  greetings.swift                              # our file!

If all goes well, you should have a libgreetings.dylib shared library in that directory.

Now, fire up a R console session in that directory and do:

greetings_lib <- dyn.load("libgreetings.dylib")

If there are no errors, the shared library has been loaded into your R session and we can use the function we just made! Let’s wrap it in an R function so we’re not constantly typing .Call(…):

greetings_from <- function(who = "me") {
  invisible(.Call("greetings_from", as.character(who[1])))
}

I also took the opportunity to make sure we are sending a length-1 character vector to the C/Swift function.

Now, say hello!

greetings_from("hrbrmstr")

And you should see:

Greetings, 🌎, it's hrbrmstr!

FIN

We’ll stop there for now, but hopefully this small introduction has shown how straightforward it can be to bridge Swift & R in the other direction.
