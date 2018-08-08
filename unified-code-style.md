# Elttob Unified Code Style

The Elttob UCS is a system for styling code in a consistent and unified way across languages and project types.

## Internationalisation

### Language
The standard language of all Elttob UCS projects is English. The particular variant of English is dependent on the project author's preferences and must be clearly stated and consistently honoured throughout the codebase, documentation and project.

If no variant is defined, the default language is British English.

### Capitalisation
The standard capitalisation method for regular text is sentence case.

## Source code

### Indentation
Indentation must always use tabs. All spaces used for indentation must be converted into tabs before being committed into a project. Any code using spaces for indentation should be rejected or converted into tabs.

The standard width of a tab character is 4, though the Elttob UCS ensures that code formatting is independent of tab width to accomodate other editors where tab width is not editable.

**Do this**
```C
// using tabs takes fewer characters, accomodates older text editors and allows variable tab width in supporting IDEs
void aFunction(int status) {
	if(status != 0) {
		return -2;
	}
	return 4;
}
```

**Do not do this**

```C
// spaces take 4x as many characters, is annoying to edit in older text editors, and is inflexible
// please kill this trend
void aFunction(int status) {
    if(status != 0) {
        return -2;
    }
    return 4;
}
```

### Alignment
Alignment should be reserved for a select few cases; these depend on the project. Most times, this is used when manipulating similar values, calling similar functions, unrolling loops, and other cases where statements are related to each other. Alignment should always be done with spaces, and must only be done within the same level of indentation. Do not apply alignment if it would create a distance of more than 8 characters between two consecutive parts of a statement. Additionaly, don't align semicolons and other small code features - since they aren't important they don't need to be separated from the rest of the statement.

**Do this**
```JavaScript
// these array accesses are related by the increasing indices
let r = imageData.data[index*4    ];
let g = imageData.data[index*4 + 1];
let b = imageData.data[index*4 + 2];
let a = imageData.data[index*4 + 3];
```

**Do not do this**
```JavaScript
// this is a stupid amount of indentation
let nodeExpressAppGet = getNodeExpressAppGet();
let cfg               = getSettings();

// this is pretty bad, too
let     i = 10		;
let     j = 5		;
{
	let k = 10 * j	;
	let l = 5  * i	;
}
```

### Comments
Prefer line comments for comments that only take up 1-3 lines, and block comments for longer comments and longer documentation. Avoid placing line comments at the end of lines - prefer placing line comments on the line before. Where extra spacing is needed to aid code readability, you may add an empty line before the start of a line or block comment.

Header comments are usually block comments that appear at the start of a file, that help document what the file does and keep track of changes. In Elttob UCS, they should only contain a description of the file, and version control such as Git should be preferred to tracking changes via header comments.

**Do this**
```Lua
--[[
	TableUtils provides helper functions for working with tables, such as measuring the length of map tables.
]]

local TableUtils = {}

--[[
	Measures the length of a map table.

	Arguments:
	t - map table to measure

	Returns:
	number - length of 't' (number of non-nil values it contains)
]]
function TableUtils.map_len(t)
	local total = 0
	for i, v in pairs(t) do
		total = total + 1
	end
	return total
end

-- return TableUtils to module manager
return TableUtils
```

**Do not do this**
```Lua
--	TableUtils.lua
--	Version history:
--		created file (John Doe)
--		updated map_len (Jane Doe)

local TableUtils = {}

--	Measures the length of a map table.
--
--	Arguments:
--	t - map table to measure
--
--	Returns:
--	number - length of 't' (number of non-nil values it contains)
function TableUtils.map_len(t)
	local total = 0
	for i, v in pairs(t) do
		total = total + 1
	end
	return total
end
--[[ return TableUtils to module manager ]]

return TableUtils
```

