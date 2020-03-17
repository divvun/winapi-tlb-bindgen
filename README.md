This crate is no longer maintained because I do not use Windows any more.

---

Generates bindings to COM interfaces, enums and coclasses.


# Usage

See the `test-msxml` and `test-wmi` subdirectories for full examples of using this library to generate bindings for the MSXML library and the WMI library respectively.

1. Find the typelib for the COM library you want to generate bindings for:

	- If you have a `.tlb` file, use that.
	- If you have a `.dll` with an embedded `.tlb` resource, use that.
	- If you have a `.idl`, generate a `.tlb` with `midl.exe` from the Windows SDK via `midl.exe .\foo.idl /tlb .\foo.tlb` and use that.

	To be sure that a `.tlb` / `.dll` will work with `winapi-tlb-bindgen`, you can create a C++ project in MSVC and try to [`#import` the `.tlb` / `.dll`.](https://docs.microsoft.com/en-us/cpp/preprocessor/hash-import-directive-cpp) If that compiles, then it should work with `winapi-tlb-bindgen`

1. Write a build script that uses this crate to generate the bindgen output for the COM library.

	```rust
	// build.rs

	winapi_tlb_bindgen::build(
		std::path::Path::new(r"C:\Program Files (x86)\Windows Kits\10\Lib\10.0.18362.0\um\x64\MsXml.Tlb"),
		false,
		out_file, // $OUT_DIR/msxml.rs
	).unwrap();
	```

1. Add an empty mod file that `include!`s the bindgen output.

	```rust
	// src/msxml.rs

	include!(concat!(env!("OUT_DIR"), "/msxml.rs"));
	```

1. Add a dependency to your crate on [`winapi = { version = "0.3.6" }`](https://docs.rs/winapi/0.3.x/x86_64-pc-windows-msvc/winapi/) You will likely want to enable (atleast) the `objbase`, `oleauto`, and `winerror` features to get access to `HRESULT`, `IUnknown` and other COM types.

1. Build your crate.

1. Silence warnings for identifier names and unused functions as necessary, and prepend imports from winapi for any types the compiler can't find.

	```rust
	// src/msxml.rs

	#![allow(non_camel_case_types, non_snake_case, unused)]

	use winapi::{ENUM, RIDL, STRUCT};
	use winapi::shared::guiddef::GUID;
	use winapi::shared::minwindef::UINT;
	use winapi::shared::winerror::HRESULT;
	use winapi::shared::wtypes::{BSTR, VARIANT_BOOL};
	use winapi::um::oaidl::{IDispatch, IDispatchVtbl, LPDISPATCH, VARIANT};
	use winapi::um::unknwnbase::{IUnknown, IUnknownVtbl, LPUNKNOWN};

	include!(concat!(env!("OUT_DIR"), "/msxml.rs"));
	```

	Repeat till there are no more missing imports and the crate compiles.

	([Issue #2](https://github.com/Arnavion/winapi-tlb-bindgen/issues/2) is about automating this step or atleast making it easier.)

1. Compare the output against [the C++ headers generated by MSVC with `#import`.](https://docs.microsoft.com/en-us/cpp/preprocessor/hash-import-directive-cpp#_predir_the_23import_directive_header_files_created_by_import) File a bug if something was emitted incorrectly.

1. Enjoy your COM API bindings.

	```rust
	// src/main.rs

	mod msxml;

	fn main() {
		unsafe {
			let hr = winapi::um::objbase::CoInitialize(std::ptr::null_mut());
			assert!(winapi::shared::winerror::SUCCEEDED(hr));

			let mut document: *mut winapi::ctypes::c_void = std::ptr::null_mut();
			let hr =
				winapi::um::combaseapi::CoCreateInstance(
					&<msxml::DOMDocument as winapi::Class>::uuidof(),
					std::ptr::null_mut(),
					winapi::um::combaseapi::CLSCTX_ALL,
					&<msxml::IXMLDOMDocument as winapi::Interface>::uuidof(),
					&mut document,
				);
			assert!(winapi::shared::winerror::SUCCEEDED(hr));
			let document = &*(document as *mut msxml::IXMLDOMDocument);

			// ...

			document.Release();
		}
	}
	```


# `winapi-tlb-bindgen-bin`

The `winapi-tlb-bindgen-bin` crate is a binary that takes in the path of the typelib as a command-line parameter, and writes the bindgen output to stdout. This can be used to generate bindings manually for greater control, as opposed to using a build script to automatically generate the bindings on every build. You would also do this if you wanted your crate to be able to be built on non-Windows platforms.

```powershell
cd winapi-tlb-bindgen-bin
cargo run -- 'C:\Program Files (x86)\Windows Kits\10\Lib\10.0.16299.0\um\x64\MsXml.Tlb'
```
