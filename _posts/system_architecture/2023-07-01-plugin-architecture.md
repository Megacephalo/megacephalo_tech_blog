---
layout: post
title: How Plugin Architecture works?
description: >
  How does Plugin Architecture works? How can I create my own plugins or mods?
sitemap: false
hide_last_modified: true
---

Have you ever wonder how the mods are developed and added into a videogame? Have you also wonder why there are so many propietary as well as open-source softwre that allows their users to add or remove plugins whenever they pleased? I also raised the same questions. Fortunately, I implemented a point cloud processing pipeline that I want to design it in a way that each step of the point cloud processing can be seen as its own plugins. 
In fact, some of the popular software are design this way, consisting of a core and the plugins. Or in the case of Eclipse in which every feature is a plugin. 


# The up-side of plugin architecture

- Flexible on development
- Flexible on usage
- Easily scalable
- Modular to extend the functionalities

# The downside of plugin architecture

- Will make your softwre slightly more complex
- Each plugin runs on its thread as a run() or execute() method.
- Once there is a change in the core, plugins would no longer work. See an actual example of [Subnautica Live At Large Udate](https://steamcommunity.com/app/264710/discussions/0/3730700312409631960/)

# Let's get our hands dirty

Let's do a workable barebone implementation of such architecture. Say I want to implement only two plugins and both of them should derive from a plugin base class that down the road we can make different types of these plugins more specialized on their traits. Here I am using C++ to exploit its various polymorphism perks, I am sure you can implement the same expression in various other OOP languages as what only differs would be the way the language manipulates dynamic memory while reading in from built libraries or modules. 
With this being said, inevitably I will explain the way you implement plugin architecture in the said language.

One thing before we dive in. The full source can be found [in my GitHub repo](https://github.com/Megacephalo/plugin_architecture_practice) just FYI.

## Overall view of our barebone program

The core consists of simply a main function to handle the calling and closing of the plugins. For real-life applications, you could adjust the role that the role play in this type of architecture, either to almost control the majority of the business logic or, to the other extreme, adopts the "The best API is no API" policy of letting all features implemented and run as plugins.

Please refer to Fig 1 for the barebone program's static class diagram.

![](/assets/img/posts/system_architecture/plugin_archittecture/plugin_architecture_barebone.png)

It seems like an ordinary inheritance. But how do we impleent this structure to become actual running plugins?

## Let's code shall we?

First, let work on our plugin base class and the related function:

### Plugin base class

| plugin_interface.h |
|--------------------|
```C++
#ifndef _PLUGIN_INTERFACE_H_
#define _PLUGIN_INTERFACE_H_

#include <iostream>
#include <memory>
#include <functional>

class PluginInterface {
public:
    virtual ~PluginInterface() {}
    virtual void run() = 0;
};

extern "C" {
    using createPlugin_t = std::shared_ptr<PluginInterface> (*)();
}

#endif /* _PLUGIN_INTERFACE_H_ */

```
In the PluginInterface class, we follows the good practice of sliding in a virtual destructor so a derived class pointer created from a base class one could properly be deleted in an orderly fashion avoiding possible memory leak.

There is an abstract virtual method that would be where the plugin would be executed and is crucial for any plugin to work in our program. This method is where the business logic of the plugin resides.

The extern "C" tells the C++ complier to use C-style linking for the code in between the braces. In C++ you can have functions of the same name but taking in different types and number of arguments. However this is not the case in C. When you dynamically load in the functions from a shared library (we will see how to do this soon), you need to "call" the function anme as string. In C++ this would be impossible as the compiler would "mangle" the function names to support function overloading. By using extern "C" you tell the compiler not to mangle the function name so it can load in the precise function by name. 

But what is this line within the braces? This is where the plugin is actulally instantiated or created. To make the most out of modern C++ smart pointers, I am replacing the C-style pointer with `shared_ptr`, however you can use whatever types of pointer you please. The pointer creates a function pointer that would need to bring in the `()` ooperator to make the magic happen. You will see how this magic unfold just in a minute.


### The plugins the plugins the plugins...

I will only showcase the implementation of a single plugin, but the other ones should follow the same vein. 

First, let's write up our header file.

```C++
#ifndef _PLUGIN_1_H_
#define _PLUGIN_1_H_

#include "plugin_interface.h"

class Plugin1 : public PluginInterface {
  public:
    Plugin1() = default;
    void run() override;

};

extern "C" {
    std::unique_ptr<PluginInterface> createPlugin() {
        return std::make_unique<Plugin1>();
    }
}

#endif /* _PLUGIN_1_H_ */
```

Usual stuff, but note the lines within the extern "C" braces. These lines actualy differ from the line in the `PluginInterface` class. This is the function that you will call by name (remember that lengthy extern "C" explanation we have in the section before?) And why is this a pointer to the base class? Because we will overload the base function with the returned instace of this function. Hence, we have all the pieces of the puzzle at hand.

### The core

I will show one way of calling the plugins to run in their separate threads. Without calling the threads, you will expect one plugin only runs after another finishes, i.e. in sequential order.

```C++
#include <iostream>
#include <string>
#include <memory>
#include <filesystem>
#include <thread>
#include <vector>
#include <dlfcn.h>
#include <functional>

#include "plugin_interface.h"
#include "plugin_1/plugin_1.h"

std::unique_ptr<PluginInterface> loadPlugin(const std::string& pluginLib) {
    void* handle = dlopen(pluginLib.c_str(), RTLD_LAZY);
    if (handle == nullptr) {
        std::cerr << "Error loading plugin: " << dlerror() << std::endl;
        return nullptr;
    }

    dlerror();  // Clear any existing error

    createPlugin_t pluginFunc = (createPlugin_t) dlsym(handle, "createPlugin");

    const char* dlsym_error = dlerror();
    if (dlsym_error) {
        std::cerr << "Cannot load symbol createPlugin: " << dlsym_error << std::endl;
        dlclose(handle);
        return nullptr;
    }

    return pluginFunc();
}

int main(int argc, char** argv) {
    std::cout << "Launchin Plugin Architecture practice" << std::endl;

    // Read all .so files under the given directory and Load all plugins (run every one of them)
    std::string pluginDir = "/home/charly/Public/plugin_architecture_practice/lib";
    std::vector<std::thread> loadPluginThreads;
    for (const auto & entry : std::filesystem::directory_iterator(pluginDir)) {
        if (entry.path().extension() != ".so") continue;

        std::unique_ptr<PluginInterface> plugin = loadPlugin(entry.path().string());
        if (plugin == nullptr) {
            std::cerr << "Error loading plugin from " << entry.path().string() << std::endl;
            continue;
        }

        // Push the function object of to load the plugin
        std::cout << "Loading a plugin from " << entry.path().string() << std::endl;
        loadPluginThreads.emplace_back(std::thread([plugin = std::move(plugin)]() {
            plugin->run();
        }));
    }

    dlerror();  // Clear any existing error

    for (auto& thread : loadPluginThreads) {
        std::cout << "Joining thread" << std::endl;
        if (thread.joinable()) thread.join();
    }

    return EXIT_SUCCESS;
}
```

Let's break the code down piece by piece. First let's look at main(). 

```C++
std::string pluginDir = "/home/charly/Public/plugin_architecture_practice/lib";
std::vector<std::thread> loadPluginThreads;
for (const auto & entry : std::filesystem::directory_iterator(pluginDir)) {
    // Let assume all read in entry is a shared library

    std::unique_ptr<PluginInterface> plugin = loadPlugin(entry.path().string());

    // Push the function object of to load the plugin
    std::cout << "Loading a plugin from " << entry.path().string() << std::endl;
    loadPluginThreads.emplace_back(std::thread([plugin = std::move(plugin)]() {
        plugin->run();
    }));
}
```
At this stage, we assume the plugins are built into their corresponding shared libraries. In POSIX machines, the shared libraries have the extension `.so`. So at this point, the program list all available shared libraries that it can read the classes and functions from. In the for loop, each plugin is read in from the shared library by name, once the plugin in read in and created using the function pointer, the plugin class' method `run()` will be run as its separate thread.

Now, let's move on to understand what the function `loadPlugin` is doing.

```C++
std::unique_ptr<PluginInterface> loadPlugin(const std::string& pluginLib) {
    void* handle = dlopen(pluginLib.c_str(), RTLD_LAZY);
    if (handle == nullptr) {
        std::cerr << "Error loading plugin: " << dlerror() << std::endl;
        return nullptr;
    }

    dlerror();  // Clear any existing error

    createPlugin_t pluginFunc = (createPlugin_t) dlsym(handle, "createPlugin");

    const char* dlsym_error = dlerror();
    if (dlsym_error) {
        std::cerr << "Cannot load symbol createPlugin: " << dlsym_error << std::endl;
        dlclose(handle);
        return nullptr;
    }

    return pluginFunc();
}
```
The steps are as follows,

1. Create a handle to the shared library. Since the program does not know what type of memory it holds, we assign the handle as a pointer to undetermined type, hence void*. The RTLD_LAZY flag means the relacation of the dynamic symbol (I would image are the underlying classes or functions) shall be done when it is actually needed and not all at once. By dynaically binding sumbols when it is actually needed should improve the performance on implementation. So let's recap the code to do this:

```C++
void* handle = dlopen(pluginLib.c_str(), RTLD_LAZY);
```

2. Create the function pointer of the plugin class. Note, at this point you are still one step behind getting the actual plugin instance. This step is done using the line:
3. 
```C++
createPlugin_t pluginFunc = (createPlugin_t) dlsym(handle, "createPlugin");
```
Remember that function you implemented inside of each plugin header file?
```C++
extern "C" {
    std::unique_ptr<PluginInterface> createPlugin() {
        return std::make_unique<Plugin1>();
    }
}
```
Now it starts to make more sense.

4. Create the actual plugin pointer instance:

```C++
return pluginFunc();
```
which will return `std::unique_ptr<PluginInterface>`, a base class pointer overloaded with the plugin you want to load in the core.

How to get the plugin to work? Let's get back to main function:

```C++
std::unique_ptr<PluginInterface> plugin = loadPlugin(entry.path().string());

plugin->run();
```

This should do the job. In the implementation of the main function, we are simply getting a step further to make this function as a thread. At the end of the day, the threads of each plugin get created and then joined.

Last but not least, let's take a look at how to compile all this jazz jusing `CMakeLists.txt`.

My project file hierarchy looks like this:

```
Plugin_architecture_practice/
├── CMakeLists.txt
├── applications
|   ├── CMakeLists.txt
|   ├── main.cpp
├── include/
|   ├── plugin_interface.h
|   ├── plugin_1/
|   |   ├── plugin_1.h
|   ├── plugin_2/
|       ├── plugin_2.h
├── src/
    ├── CMakeLists.txt
    ├── plugin_1/
    |   ├── CMakeLists.txt
    |   ├── plugin_1.cpp
    ├── plugin_2/
        ├── CMakeLists.txt
        ├── plugin_2.cpp
```

The file structure looks kind of formidable but the underlying idea is to have each plugin built as its individual shared library, and later on have whatever application, where the main function lives, call in the necessary plugin libraries.

First, let take a look at the project root CMakeLists.

| CMakeLists.txt |
|----------------|
```cmake
cmake_minimum_required(VERSION 3.2)

project(Plugin_Architecture_Practice)

set (CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_FLAGS "-Wall")
set(CMAKE_BUILD_TYPE Debug)

set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}/lib)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}/lib)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}/bin)

include_directories(include)

# Build plugins
add_subdirectory(src)

# Build applications
add_subdirectory(applications)
```
I prefer to build and place the deliverables in their dedicated directories, so you are free to choose whether you want to use the lines:

```cmake
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}/lib)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}/lib)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}/bin)
```
Now, move on to the plugin's CMakeLists.txt

```cmake
# Build a dynamic library
add_library(plugin_1 SHARED 
    plugin_1.cpp
)
```
In this case, after the compilation you should expect a `libplugin_1.so` file living under your designated directory. Mine is under `lib/`.

Now it is time to load the plugin library to the application of your choice.

```cmake
add_executable(${PROJECT_NAME}_node
    main.cpp
)
target_link_libraries( ${PROJECT_NAME}_node
    plugin_1
)
```
That's it! For the very first time, compile the whole CMake project using the command:

```bash
$ mkdir build
$ cd build
$ cmake ..
$ make
```
For subsequent compiling, you just need to __cd__ to your `build/` directory and insert the command `make`.

# Conclusion

This is all about it! Congratulations! You just learnedd how to implement a plugin architecture the modern C++ way.There are still a couple places that I can observe possible improvements.

## Room for improvement

One downside of using C and C++ in implementing programs involving manipulating dynamic memory is that we still require the use of C-style pointer as in the case of `void*` which without proper handling can lead to unoticeable memory leak. Unfortunately C/C++ would not explicitly yell at you upfront that there is any issue or caveats looming in the dark, so it is up to the programmer to take care of each possible path to manipulating the memory and releasing it afterwards. 

Speaking of releasing memory, there is clearly an issue in this code. Do you remember that we use `dlopen()` to return a void* poiner? The program never releases the resource under normal procedure with `dclose(hadle)`. This should be taken care of by the programmer.

Moreover, by manipulating the dynamic memory, profiling the memory usage with tools such as **Valgrind** may sometimes yield an existence of memory leakage even though the resource might long being released. With this being said, we have not done dclose() yet so this is not the case.

In the same fasion, despite using smart pointer to wrapp the function pointer returned from `dlsym()`, it is still subject to potential memory leakage. To solve this issue would require uninstinctive solution to either mitigate, solve, or get around the issue. I personally find either way hideous.

Anyways, this is the fun of programming I guess...I hope this article helps you should you ever want to implement a plugin architecture of your own. Keep calm and keep coding. Peace!

# References

- [Joydip Kanjilal - Why do we Need Virtual Destructors? - LinkedIn](https://www.linkedin.com/pulse/why-do-we-need-virtual-destructors-joydip-kanjilal)

- [How To Create a Flexible Plug-In Architecture? - StackOverflow](https://stackoverflow.com/questions/2768104/how-to-create-a-flexible-plug-in-architecture)

- [Omar Elgabry - Plug-in Architecture - Medium](https://medium.com/omarelgabrys-blog/plug-in-architecture-dec207291800)

- [Plugin Architecure - retrieved file type: PDF](https://cs.uwaterloo.ca/~m2nagapp/courses/CS446/1195/Arch_Design_Activity/PlugIn.pdf)

- [Joseph Gefroh - How to Design Software — Plugin Systems - Medium](https://betterprogramming.pub/how-to-design-software-plugins-d051ce1099b2)

- [ahota/zoo: An example C++ plugin architecture with on-the-fly plugin inclusion  - GitHub](https://github.com/ahota/zoo)