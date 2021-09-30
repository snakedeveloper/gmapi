# Note: This project is no longer supported, repo is for archival purposes only
***

![GMAPI](https://github.com/snakedeveloper/gmapi/blob/main/gmapi.png?raw=true)

# About GMAPI

GMAPI is a programming library for Game Maker 6/7/8 DLL developers, that gives the possibility of accessing the game's resources right from the DLL! What it currently features:

- Ability to use the GML functions within your DLL.
- Possibility to modify or even register a new GML functions for the game (increases external calling efficiency)
- Direct access to the game resources, such as:
- Sprite, background and font bitmaps
- Sound files
- Local and global variables/arrays
- Built-in variables/properties (e.g room_speed, fps)
- Internal game structures (e.g. the one that holds GM instance's data)
- Game's Direct3D interfaces (IDirect3D8, IDirect3DDevice8, IDirect3DTexture8)

In the current version you have a possibility to access sprites, backgrounds, surfaces, sounds, scripts, particles, fonts and some other data, such as e.g. game's Direct3D interfaces, window handles, GML functions, GM version and more. Soon I'll try to give you possibility to access other runner's data as well... I'm also thinking about adding new features like adding a sprite to game from bitmap loaded in memory. Well, you can give suggestions what could be added to the library (BTW. something important/useful related to DirectX will be welcomed - I don't know much about that library ;p). Unfortunatelly, because of lack of time i didn't wrote a documentation... there are only some examples & comments in a header file... here i'll explain most important things.
Library is released on LGPL license, so everyone can take a look at the source and check out how it is made, or modify... :(
When project will be in more advanced stage (and there will be time for this ofc) i'll make the library compatible with MinGW compiler ;o

# Installation

All you have to do is extract the archive to chosen directory and specify the paths to "include" & "lib" directories in IDE. Run Visual C++, select from "Tools" menu "Options..." item, next go to "Projects and Solutions" >> "VC++ Directories" node, now select "Include files" from the "Show directories for" combobox and add the path to "include" directory from GMAPI, then select "Library files" from the combobox and add the path to "lib" directory. Done.

# Initialization

Before you'll begin to work with the library you must include gmapi.h header to one of the source files in your project. If you don't have DirectX SDK installed define GMAPI_NO_D3D before including gmapi.h (D3D interfaces will be defined as void), thus this will look like:

```
#define GMAPI_NO_D3D
#include <gmapi.h>
```
Now you must add to the library dependencies proper static library - all depends on which runtime library you choose in your project:

- gmapi-mt.lib - Multithreaded
- gmapi-mt-dll.lib - Multithreaded DLL
- gmapi-mt-d.lib - Multithreaded debug
- gmapi-mt-d-dll.lib - Multithreaded debug DLL

Next, you must create main library object, that is CGMAPI class. That class is a singleton, so only one instance can be created at runtime in your library. To create an instance of this class use CGMAPI::Create static method, which takes as parameter a pointer to "unsigned long" variable type - it'll be used to return the result of initialization (possible values are: GMAPI_INITIALIZATION_SUCCESS - if initialization succeeded; GMAPI_INITIALIZATION_FAILED - when initialization fails; GMAPI_ALREADY_INITIALIZED - when an instance of CGMAPI class already exists). The method returns pointer to newly created class object, which you should store in a variable. I recommend to do this in DLL's entrypoint, that is DllMain function, for example:
```
gm::CGMAPI* gmapi;

BOOL WINAPI DllMain( HINSTANCE aModuleHandle, int aReason, int aReserved ) {
  switch ( aReason ) {
    case DLL_PROCESS_ATTACH: {
      // Initialization
      unsigned long result = 0;
      gmapi = gm::CGMAPI::Create( &result );

      // Check the result
      if ( result != gm::GMAPI_INITIALIZATION_SUCCESS ) {
        MessageBox( 0, "Unable to initialize GMAPI.", 0, MB_SYSTEMMODAL | MB_ICONERROR );
        return FALSE;
      }

      break;
    }

    case DLL_PROCESS_DETACH:
      // Release from memory & deinitialize GMAPI
      gmapi->Destroy();
      break;
  }

  return TRUE;
}
```

CGMAPI::Destroy() method frees the class instance from memory. BTW, all classes, functions, constants etc. in GMAPI are declared in "gm" namespace. You should not call any GML functions immediatelly after initialization - GMAPI hooks external_call function in runner in order to retrieve pointer to current instance (that is, from which the function is called) which is needed to call all GM functions properly in DLL. So, it is necessary to return to GM after initialization. When class object has been created without any problems you can call the GML functions from within your DLL and, of course, you can use CGMAPI class components.

# Calling GM functions

When GMAPI is initialized you can now call GM functions from the DLL. All functions are nicely wrapped in the library so you can call them in a simple way, take a look:

```
int list;
gm::CGMVariable value;

list = gm::ds_list_create();

gm::ds_list_add( list, "test" );
gm::ds_list_add( list, 12345.12345 );

value = gm::ds_list_find_value( list, 0 ); // store "test" in the variable
value = gm::ds_list_find_value( list, 1 ); // return 12345.12345 to the variable

gm::ds_list_destroy( list );
```

CGMVariable class object that is defined in this code snippet mimics the GM variable, so it can be set to either string or real value. It must be used to store return values from functions that can return values that type is not known (e.g. ds_list_find_value can return either string or real). Also, you must pass it in parameters in some GM functions for the same reason... besides, that class emulates string type from Delphi - it deals with string allocation, "conversion" from char*, deallocation etc... using runner - char* is not fully compatible with Delphi string ("length" variable, reference counter for the garbage collector) so i had to do something ^^ Oh, one more thing... you should always return the result of GM function that can return string to the CMGVariable class object - it'll take care of deallocation.
All GM functions are available in GMAPI, excluding math functions and those that are operating on strings (and functions such as is_real or is_string). In addition, class has some constructors overloaded and it can be instatiated implicitly by passing a value in parameters that type is a reference to a CGMVariable class (e.g. ds_list_add takes CGMVariable reference type in second parameter, in above code snippet there will be created an temporary instance of class by passing "char*" or "double" to the function). There are also some operator overloading implemented, like "++", "-=" or casting to double/char*, plus "<<" overload for std::ostream :)

# Accessing game resources

You can access them by a three ways: with GM functions of course (least efficent, less possibilities), by GM's structures that I've analyzed in runner and with "accessing classes" defined in GMAPI. Using structures is most efficent and equally unsafe, I'll explain how to use them in a documentation that I'll write later. Meanwhile, you can use the classes I mentioned - they're safer and easier to use than using the GM's structures. Why I called them "accessing" (wtf) ? Well, they're operating entirely on runner's memory, what is also reason why using them is somewhat entangled. In CGMAPI class there are some objects of these classes defined: Sprites (ISprites class), Backgrounds (IBackgrounds), Sounds (ISounds), Surfaces (ISurfaces) and Scripts (IScripts). All of these classes shares some methods that can retrieve various data that is relevant to respective GM resource type (for example, GetID() returns an ID of specified resource by given name of this type, GetCount() returns number of resources of this type) and all of them have an array operator ("[]") overloaded that returns reference type to next "accessing classes" - these classes gives you access to resource with ID specified in brackets. Damn, hard to explain... so, for example:

```
gm::CGMAPI* gmapi;
(...)
// Let's say that we've added an animated sprite named spr_anim
// to the game, now we want to get width of the sprite
int sprAnim, spriteWidth;

// First, we must get the ID
sprAnim = gmapi->Sprites.GetID( "spr_anim" );

// Now we will get the width
spriteWidth = gmapi->Sprites[sprAnim].GetWidth();

// What if our sprite has not been found because we didn't added it
// to the game ? An exception with EGMAPISpriteNotExists class object will be thrown
// (exception class), we will catch it with "try - catch" of course
try {
  spriteWidth = gmapi->Sprites[sprAnim].GetWidth();
}
catch ( gm::EGMAPISpriteNotExist& exc ) {
  exc.ShowError(); // Show message with information about error
}

// Operator [] in ISprites class returns ISprite reference type - that class is
// the only one of these "higher level" that have defined object of next such
// class - ISpriteSubimages (Subimages object), which gives you access to specified
// sprite subimage. Hm, let's say we want to store all of the texture ID (a'la
// sprite_get_texture from GM) in a vector array:
std::vector<int> textures;

for ( int i = 0; i < gmapi->Sprites[sprAnim].Subimages.GetCount(); i++ )
  textures.push_back( gmapi->Sprites[sprAnim].Subimages[i].GetTextureID() );
```
