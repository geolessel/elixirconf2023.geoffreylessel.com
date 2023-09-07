---
theme: seriph
background: /pinnacle.jpg
class: text-center
highlighter: shiki
lineNumbers: true
info: |
  ## Introducing Vox
  Slide deck for ElixirConf 2023
drawings:
  persist: false
title: ElixirConf 2023 - Introducing Vox
mdc: true
---

# Introducing VOX

## The static site generator for Elixir lovers

<div class="abs-br m-6 flex gap-2">
  <a href="https://github.com/geolessel/vox" target="_blank" alt="GitHub"
    class="text-xl slidev-icon-btn opacity-80 !border-none !hover:text-white">
    <carbon-logo-github />
    geolessel/vox
  </a>
</div>




---
transition: slide-left
layout: image-right
image: /geo.jpg
---

# Geoffrey Lessel

<v-clicks>

- Elixir lover for 9 years
- Odd-year ElixirConf alum (spoke at 2017, 2019, 2021, and now 2023)
- Blog(ged) at https://geoffreylessel.com
- Wrote **Phoenix in Action** for Manning http://phoenixinaction.com (use code `PHX23`)
- Made **Build It With Phoenix** https://builditwithphoenix.com (use code `elixirconf`)
- Various other Elixir shenanigans

</v-clicks>

---
layout: fact
transition: slide-up
---

# How it started




---
layout: two-cols-header
transition: slide-down
---

# BuildItWithPhoenix.com

::left::

# Things I wanted

<v-clicks>

- Elixir
- HTML
- mirrored directory structure
- EEx
- templates
- Tailwind
- extensible
- lightly held opinions

</v-clicks>

::right::

<v-click>

# Things I looked at

</v-click>

<v-clicks>

- jekyl
  - too much Ruby
- 11ty, nextjs, gatsby
  - too much JavaScript
- still-ex
  - too much Slime
  - archived on July 6
- lambdapad
  - too much Erlang

</v-clicks>

<style>
.slidev-vclick-hidden {
  display: block;
}
</style>

---
layout: statement
transition: slide-up
---

# How it went



---
layout: statement
transition: slide-up
---

# A brief history



---
layout: center
transition: slide-left
---

# 2019: I wanted to sim race

## Result?


---
layout: center
transition: slide-right
---

# External steering wheel display

![](/elixirconf-2019.png)





---
layout: center
transition: slide-left
---

# 2020: I wanted to make music

## Result?


---
layout: center
transition: slide-right
---
# MIDI sequencer

![](/empex-2020.png)





---
layout: center
transition: slide-left
---

# 2021: I wanted to learn assembly

## Result?

---
layout: center
transition: slide-right
---
# ex6502

![](/elixirconf-2021.png)



---
layout: fact
transition: slide-down
---

You should really check out https://justforfunnoreally.dev




---
layout: statement
transition: slide-up
---

# Let's look at some code




---
layout: two-cols-header
clicks: 2
transition: fade
---

# How it works - `FileFinder`

- Gather a list of files to compile

::left::

<v-clicks>

- find _all_ the files in the `root_dir`
- add them all to a collection of unprocessed files for later processing

</v-clicks>

::right::

```elixir {all|4,10-14|5-7} {lines:true,at:0}
defmodule Vox.Builder.FileFinder do
  def collect(root_dir) do
    root_dir
    |> find()
    |> Enum.each(
      &Vox.Builder.Collection.add(&1, :unprocessed)
    )
  end

  def find(root_dir) do
    [root_dir, "**", "*"]
    |> Path.join()
    |> Path.wildcard()
  end
end
```




---
layout: two-cols-header
transition: fade
---

# How it works - `FileFinder`

- Gather a list of files to compile

::left::

- create a `File{}` struct

::right::

```elixir
defmodule Vox.Builder.File do
  @src_dir Application.compile_env(:vox, :src_dir)

  defstruct bindings: [],
            collections: [],
            compiled: nil,
            content: "",
            destination_path: "",
            final: "",
            root_template: "#{@src_dir}/_root.html.eex",
            source_path: "",
            template: "",
            type: nil
end
```





---
transition: fade
---

# How it works - `FileCompiler`

- *waves hands indiscriminately* Compile the files

```elixir
defmodule Vox.Builder.FileCompiler do
  def compile() do
    Vox.Builder.Collection.list_files()
    |> compile_files()
    |> compute_collections()
    |> compute_bindings()
    |> update_collector()
    |> eval_files()
    |> put_nearest_template()
    |> insert_into_template()
    |> update_collector()
  end
  ...
end
```




---
transition: fade
---

# `FileCompiler.compile_files/1`

```elixir {all|2-3|4-20|4,5|4,7-10|4,12-20|22-23|25-27|all} {maxHeight:'400px'}
defp compile_files(files) do
  Enum.reduce(files, [], fn %File{source_path: path} = file, acc ->
    case Path.extname(path) do
      ".eex" ->
        compiled = EEx.compile_file(path)

        destination_path =
          path
          |> Path.rootname(".eex")
          |> String.trim_leading(Application.get_env(:vox, :src_dir) <> "/")

        [
          %File{
            file
            | compiled: compiled,
              destination_path: destination_path,
              type: :evaled
          }
          | acc
        ]

      "" ->
        acc

      _ext_of_passthrough ->
        destination_path = String.trim_leading(path, Application.get_env(:vox, :src_dir) <> "/")
        [%File{file | destination_path: destination_path, type: :passthrough} | acc]
    end
  end)
end
```






---
transition: fade
---

# `FileCompiler.compute_collections/1`

```elixir {all} {maxHeight:'400px'}
iex(1)> EEx.compile_string("<% elixirconf = :fantastic %>")
{:__block__, [],
 [{:=, [line: 1], [{:elixirconf, [line: 1], nil}, :fantastic]}, {:<<>>, [], []}]}
```

<v-click at="0">

```elixir {all|1-3|4-6|7-15|8-11|13-14|17|all} {maxHeight:'300px'}
defp compute_collections(files) do
  Enum.map(files, fn file ->
    collections =
      file.compiled
      |> Macro.prewalker()
      |> Enum.map(& &1)
      |> Enum.reduce([], fn
        {:=, _line, [{:collections, _collections_line, _nil}, collections]}, acc ->
          collections = List.wrap(collections)
          Vox.Builder.Collection.add_collections(collections)
          collections ++ acc

        _other, acc ->
          acc
      end)

    %{file | collections: collections}
  end)
end
```

</v-click>





---
transition: fade
---

# `FileCompiler.compute_bindings/1`

```elixir {all}
iex(3)> Code.eval_quoted(compiled)
{"", [elixirconf: :fantastic]}
```

<v-click at="0">

```elixir {all|2|4-17|5-10|13-16|all} {maxHeight:'400px'}
defp compute_bindings(files) do
  assigns = Vox.Builder.Collection.assigns()

  Enum.map(files, fn file ->
    {_content, bindings} =
      Code.eval_quoted(
        file.compiled,
        [assigns: assigns],
        __ENV__
      )

    # make the bindings accessible through map dot notation
    bindings
    |> Enum.reduce(%{file | bindings: bindings}, fn {key, value}, file ->
      Map.put_new(file, key, value)
    end)
  end)
end
```

</v-click>




---
transition: fade
---

# How it works - `FileCompiler`

```elixir {all|3-6|7,11|8}
defmodule Vox.Builder.FileCompiler do
  def compile() do
    Vox.Builder.Collection.list_files()
    |> compile_files()
    |> compute_collections()
    |> compute_bindings()
    |> update_collector()
    |> eval_files()
    |> put_nearest_template()
    |> insert_into_template()
    |> update_collector()
  end
  ...
end
```



---
transition: fade
---

# `FileCompiler.eval_files/1`

```elixir {all|2|4-6,17|4,8-16,17|all} {maxHeight:'400px'}
defp eval_files(files) do
  collection_assigns = Vox.Builder.Collection.assigns()

  Enum.map(files, fn
    %File{type: :passthrough} = file ->
      file

    %File{type: :evaled, compiled: compiled} = file ->
      {content, _bindings} =
        Code.eval_quoted(
          compiled,
          [assigns: collection_assigns ++ file.bindings],
          __ENV__
        )

      %{file | content: String.trim(content)}
  end)
end
```



---
transition: fade
---

# `FileCompiler.put_nearest_template/1`

```elixir {all|1-3|5|7-19|9,13-15|9,10-11|23-34|26-28,33|26,30-33|21} {maxHeight:'400px'}
defp put_nearest_template(files) when is_list(files) do
  Enum.map(files, &put_nearest_template/1)
end

defp put_nearest_template(%File{type: :passthrough} = file), do: file

defp put_nearest_template(%File{source_path: path, bindings: bindings} = file) do
  template =
    case bindings[:template] do
      nil ->
        find_template_in_this_and_parent_directory({:ok, Path.dirname(path)})

      template ->
        {:ok, template} = Path.safe_relative(Path.join(Path.dirname(path), template))
        template
    end

  %{file | template: template}
end

defp find_template_in_this_and_parent_directory(:error), do: "site/_template.html.eex"

defp find_template_in_this_and_parent_directory({:ok, path}) do
  this_template = Path.join([path, "_template.html.eex"])

  case Elixir.File.exists?(this_template) do
    true ->
      this_template

    false ->
      parent_dir = Path.safe_relative(Path.join(path, ".."))
      find_template_in_this_and_parent_directory(parent_dir)
  end
end
```




---
transition: fade
---

# `FileCompiler.insert_into_template/1`

```elixir {all|2-4|2,6-21|2,6,15-16|2,6,18|2,6,20-21} {maxHeight:'400px'}
defp insert_into_template(files) do
  Enum.map(files, fn
    %File{type: :passthrough} = file ->
      file

    %File{content: content, template: template} = file ->
      bindings =
        file.bindings
        |> Enum.filter(fn
          {{_, EEx.Engine}, _} -> false
          _ -> true
        end)
        |> Enum.into(Keyword.new())

      templated =
        EEx.eval_file(template, assigns: Keyword.merge(bindings, inner_content: content))

      assigns = Keyword.merge(bindings, inner_content: templated)

      final = EEx.eval_file(file.root_template, assigns: assigns)
      %{file | final: final}
  end)
end
```



---
transition: slide-down
---

# How it works - `FileWriter`

- it writes files using `Mix.Generator`



---
layout: statement
transition: slide-left
---

# Generating a new project

## `mix vox.new`



---
transition: fade
---

# `mix vox.new`

```elixir {all|5} {maxHeight:'400px'}
defmodule Mix.Tasks.Vox.New do
  @template_string_to_replace "APP"

  use Mix.Task
  use VoxNew.Templater

  alias VoxNew.Project

  template("mix.exs")
  template("README.md")
  template("assets/app.js", :esbuild)
  template("config/config.exs")
  template("lib/application.ex", :esbuild)
  template("lib/#{@template_string_to_replace}/esbuild.ex", :esbuild)
  template("lib/#{@template_string_to_replace}.ex")
  template("site/_root.html.eex")
  template("site/_template.html.eex")
  template("site/index.html.eex")
  template("site/posts/hello-world.html.eex")
  template("test/test_helper.exs")
  template("test/#{@template_string_to_replace}_test.exs")

  @impl Mix.Task
  def run(argv) do
    {flags, [path | _rest]} = OptionParser.parse!(argv, strict: [esbuild: :boolean])

    # [TODO] I think these could result in incorrect formatting
    module_name = Macro.camelize(path)
    app_name = Macro.underscore(path)
    esbuild = Keyword.get(flags, :esbuild, false)

    generate(%Project{
      app_name: app_name,
      base_path: path,
      esbuild: esbuild,
      module_name: module_name
    })
  end

  defp generate(project) do
    project
    |> copy_templates()
  end

  defp copy_templates(%Project{} = project) do
    @templates
    |> Enum.each(fn
      {template, true} ->
        copy_template(template, project)

      {template, flag} when is_atom(flag) ->
        if Map.get(project, flag) do
          copy_template(template, project)
        end
    end)

    project
  end

  defp copy_template(template, %Project{base_path: base_path} = project) do
    contents = render_template(template, project: project)

    template_for_app =
      String.replace(template, @template_string_to_replace, project.app_name)

    write_path = Path.join(base_path, template_for_app)
    Mix.Generator.create_file(write_path, contents)
  end
end
```




---
transition: fade
---

# The magic of macros

```elixir {all|2-8|5,25-29|6,10-23|10,13,15|10,13,17-21} {maxHeight:'400px'}
defmodule VoxNew.Templater do
  defmacro __using__(_env) do
    quote do
      import unquote(__MODULE__)
      Module.register_attribute(__MODULE__, :templates, accumulate: true)
      @before_compile unquote(__MODULE__)
    end
  end

  defmacro __before_compile__(env) do
    base_path = Path.expand("../../templates", __DIR__)

    for {template_path, _flag} <- Module.get_attribute(env.module, :templates) do
      path = Path.join(base_path, template_path)
      compiled = EEx.compile_file(path)

      quote generated: true do
        @external_resource unquote(path)
        @file unquote(path)
        def render_template(unquote(template_path), var!(assigns)), do: unquote(compiled)
      end
    end
  end

  defmacro template(name, flag \\ true) do
    quote do
      @templates {unquote(name), unquote(flag)}
    end
  end
end
```



---
transition: slide-down
---

# `mix vox.new`

```elixir {all|2,9-21|23-38|32,40-43|42,45-58|47-49|47,51-54|49,53,60-68|60,61,68|60,63-67,68} {maxHeight:'400px'}
defmodule Mix.Tasks.Vox.New do
  @template_string_to_replace "APP"

  use Mix.Task
  use VoxNew.Templater

  alias VoxNew.Project

  template("mix.exs")
  template("README.md")
  template("assets/app.js", :esbuild)
  template("config/config.exs")
  template("lib/application.ex", :esbuild)
  template("lib/#{@template_string_to_replace}/esbuild.ex", :esbuild)
  template("lib/#{@template_string_to_replace}.ex")
  template("site/_root.html.eex")
  template("site/_template.html.eex")
  template("site/index.html.eex")
  template("site/posts/hello-world.html.eex")
  template("test/test_helper.exs")
  template("test/#{@template_string_to_replace}_test.exs")

  @impl Mix.Task
  def run(argv) do
    {flags, [path | _rest]} = OptionParser.parse!(argv, strict: [esbuild: :boolean])

    # [TODO] I think these could result in incorrect formatting
    module_name = Macro.camelize(path)
    app_name = Macro.underscore(path)
    esbuild = Keyword.get(flags, :esbuild, false)

    generate(%Project{
      app_name: app_name,
      base_path: path,
      esbuild: esbuild,
      module_name: module_name
    })
  end

  defp generate(project) do
    project
    |> copy_templates()
  end

  defp copy_templates(%Project{} = project) do
    @templates
    |> Enum.each(fn
      {template, true} ->
        copy_template(template, project)

      {template, flag} when is_atom(flag) ->
        if Map.get(project, flag) do
          copy_template(template, project)
        end
    end)

    project
  end

  defp copy_template(template, %Project{base_path: base_path} = project) do
    contents = render_template(template, project: project)

    template_for_app =
      String.replace(template, @template_string_to_replace, project.app_name)

    write_path = Path.join(base_path, template_for_app)
    Mix.Generator.create_file(write_path, contents)
  end
end
```




---
layout: statement
transition: slide-up
---

# Did it work?

## Demos






---
layout: two-cols-header
transition: slide-down
---

# Where do things stand?

::left::

## Working

<v-clicks>

- installer (`mix vox.new`)
- esbuild
- `mix vox.build`
- `mix vox.dev`

</v-clicks>

::right::

<v-click>

## Needs work

</v-click>

<v-clicks>

- annoying warnings
- better errors
- TESTS
- other processor plugins
  - Markdown
- general refactoring

</v-clicks>

<style>
.slidev-vclick-hidden {
  display: block;
}
</style>



---
layout: statement
transition: fade-out
---

# VOX needs YOU



---
transition: fade-out
---

# Codes and such

## Build It With Phoenix

- https://BuildItWithPhoenix.com
- code `elixirconf` gets 15% off through next Sunday
- sign up for the mailing list - **I'M GIVING AWAY 2 COPIES NEXT WEEK**

## Phoenix in Action

- http://PhoenixInAction.com
- code `PHX23` gets you 30% off anything at Manning.com
- sign up for the Build It With Phoenix mailing list -- **I'M GIVING AWAY 2 EBOOKS NEXT WEEK**




---
transition: fade-out
layout: two-cols
---

# Find me and VOX online

- https://github.com/geolessel/vox
- @geolessel
- @geo (elixirforum.com)
- https://geoffreylessel.com
- https://builditwithphoenix.com
- http://phoenixinaction.com
