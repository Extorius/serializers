# Using Serializers
## 1. What is a serializer?
A serializer (in this context) converts data to a human-readable format, typically the same format used while writing the data as constants.

#### Example output from a serializer:
```lua
-- Input:
local t = {}
for i = 1, 10 do
    table.insert(t, string.char(i + 96))
end

t[11] = {
    [1] = 'child table'
}

-- Output:
local serialized = {
        [1] = "a",
        [2] = "b",
        [3] = "c",
        [4] = "d",
        [5] = "e",
        [6] = "f",
        [7] = "g",
        [8] = "h",
        [9] = "i",
        [10] = "j",
        [11] = {
                [1] = "child table"
        }
}
```

## 2. What are serializers used for?
A serialize has many use-cases in programming. Notably:

#### Debugging
While debugging programs, you may encounter unexpected data, or the way you handle the data might be faulty or not as you expected.

A good way of verifying the data being handled is by dumping it through serializing it.

#### Dumping
If you ever need to dump the contents of a variable for reasons outside of debugging (such as letting users view the contents of variables during runtime, or saving contents of variables to another location or file), then serializing is something you should use.

#### Recreating
For more hands-on programs, being able to view and recreate data being sent or received from an outside source (such as an API, or data being sent to remote events) is common, and serializing said data is typically the approach used.

## 3. How do serializers work?
Generally serializers are composed of a function which uses the following steps:
1. Data Types - Each piece of data sent to a serializer needs to be categorized into different types; such as strings, numbers, tables, booleans, etc.
2. To String Formating - After the data type has been established, unique string formating is applied to different types of data, to ensure it's being displayed as similar as possible to how it would be as a constant.

## 4. How can I make a serializer?
There is no set way of creating a serializer (although the general theory is essentially universal) however I personally find this to be the simplest (while still effective) method:

```lua
local serializer = {}
setmetatable(serializer, {
    __call = function(self, data, level)
        -- Using metatables allows us to use the 'self' keyword, which I find preferable to self-calling functions
        -- The data parameter is the data that will be serialized
        -- The level parameter is used for table serializing, and is used for proper indenting
    end
})
```

After the block of the function is established, we can begin sorting data into types:
```lua
local serializer = {}
setmetatable(serializer, {
    __call = function(self, data, level)
        -- Using metatables allows us to use the 'self' keyword, which I find preferable to self-calling functions
        -- The data parameter is the data that will be serialized
        -- The level parameter is used for table serializing, and is used for proper indenting
        
        local datatype = type(data) -- Assigning a variable instead of repeatedly calling the 'type' function is more optimized
        
        if datatype == 'string' then
            return string.format("%q", data) -- The string.format function in Lua automatically applies the correct quotations around a string
        elseif datatype == 'number' then
            return tostring(data) -- The 'tostring' function converts data to strings, including numbers
        elseif datatype == 'boolean' then
            if data == true then -- This approach is slightly more optimized then string.format
                return 'true'
            elseif data == false then
                return 'false'
            end
        elseif datatype == 'nil' then
            return 'nil'
        elseif datatype == 'table' then
            level = level or 1 -- Assigning the first level of indentations
            
            local result = (level == 1 and 'local serialized = ' or '') .. '{\n' -- Here, we define a local variable to the output if it's the first level
            for i,v in pairs(data) do
                if type(v) ~= 'table' then
                    result = result .. string.rep("\t", level) .. string.format('[%s] = %s,\n', self(i), self(v)) -- Serializing the index and value and using the string.format function to format it as [index] = value
                else
                    result = result .. string.rep("\t", level) .. string.format('[%s] = %s,\n', self(i), self(v, level + 1)) -- Adding to the level if the current value type is a table
                end
            end
            
            result = string.sub(result, 0, #result - 2) .. '\n' .. string.rep('\t', level - 1) .. '}' -- Removing the final newline and comma, and then adding the appropriate indents
            return result
        end
    end
})
```

#### Calling this serializer
```lua
print(serializer({
    [1] = 'Hello world!'
}))
```

#### Further optimizing
Optimizing further is essential for large amounts of data, or frequent calls to the serializer function.

To optimize, consider the following approach:
```lua
local sub = string.sub
local format = string.format
local pairs = pairs
local rep = string.rep
local tostring = tostring
local type = type
local setmetatable = setmetatable

-- Indexing globals is slower then just defining the globals as local variables; as instead of indexing the global environment, the VM can instead index the stack.

local serializer = {}
setmetatable(serializer, {
    __call = function(self, data, level)
        -- Using metatables allows us to use the 'self' keyword, which I find preferable to self-calling functions
        -- The data parameter is the data that will be serialized
        -- The level parameter is used for table serializing, and is used for proper indenting

        local datatype = type(data) -- Assigning a variable instead of repeatedly calling the 'type' function is more optimized

        if datatype == 'string' then
            return format("%q", data) -- The string.format function in Lua automatically applies the correct quotations around a string
        elseif datatype == 'number' then
            return tostring(data) -- The 'tostring' function converts data to strings, including numbers
        elseif datatype == 'boolean' then
            if data == true then -- This approach is slightly more optimized then string.format
                return 'true'
            elseif data == false then
                return 'false'
            end
        elseif datatype == 'nil' then
            return datatype -- Instead of a new constant, the VM can index the stack
        elseif datatype == 'table' then
            level = level or 1 -- Assigning the first level of indentations

            local result = (level == 1 and 'local serialized = ' or '') .. '{\n' -- Here, we define a local variable to the output if it's the first level
            for i, v in pairs(data) do
                if type(v) ~= 'table' then
                    result = result .. rep("\t", level) .. format('[%s] = %s,\n', self(i), self(v)) -- Serializing the index and value and using the string.format function to format it as [index] = value
                else
                    result = result .. rep("\t", level) ..
                                 format('[%s] = %s,\n', self(i), self(v, level + 1)) -- Adding to the level if the current value type is a table
                end
            end

            result = sub(result, 0, #result - 2) .. '\n' .. rep('\t', level - 1) .. '}' -- Removing the final newline and comma, and then adding the appropriate indents
            return result
        end
    end
})
```

## Benchmarks
#### Info
Each benchmark was done with `100000` calls to each serializer function. The parameter passed was a table with 10 objects in it:

```lua
local t = {}
for i = 1, 10 do
    table.insert(t, i)
end

for i = 1, 4 do
    local start = os.clock()
    for i = 1, 100000 do
        local serialized = serializer(t)
    end

    print(os.clock() - start)
end
```

#### Results
- *Results are in seconds*

![benchmarks|300x100](https://i.e-z.host/zyd2bjfxdh8ty6d0otiqc2z2df7d78%E2%80%8E)
