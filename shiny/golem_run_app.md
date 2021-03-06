Deploying apps with {golem}’s `run_app()`
================
September 04, 2020

When deploying [{golem}](https://github.com/ThinkR-open/golem) Shiny
applications, the `run_app()` function is very critical but can be under
appreciated/utilized (at least by me) for those new go {golem}. Here are
some examples of how I have utilized `run_app()` in my apps.

# Specifying the server port

One handy customization of the `run_app()` function is the following,
which allows you to specify what port to launch your application. In the
following example, a default port number of 9999 is set, but this can be
overridden when invoking the `run_app()` function. Personally, I like
this because it allows me to bookmark 127.0.0.1:9999 in my browser, and
quickly open up my application during development. This could also be
useful in Docker deployments where the app port needs to be explicitly
exposed (therefore, needs to be consistent).

``` r
run_app <- function(
    port = 9999,
    ...
) {
    with_golem_options(
        app = shinyApp(ui = app_ui, 
                       server = app_server, 
                       options = list(port = port)), 
        golem_opts = list(...)
    )
}
```

# Parameterizing your application deployment

Some applications may benefit from having a parameter that specifies the
deployment or some other runtime options. For example, a `view`
parameter can be added to allow for customizing the application view at
runtime.

``` r
run_app <- function(
    module = c("complete", "minimal"),
    ...
) {
    
    with_golem_options(
        
        # Using switch() to parameterize the app runtime
        app = switch(
            module,
            "complete" = shinyApp(
                ui = app_complete_ui, 
                server = app_complete_server
            ),
            "minimal" = shinyApp(
                ui = app_minimal_ui, 
                server = app_minimal_server
            )
        ), 
        golem_opts = list(...)
    )
}
```

This can also be helpful for loading in the correct configuration using
the `config` package, which you can then pass to `golem_opts` and access
throughout your app using `golem::get_golem_options()`.

``` r
# run_app.R
run_app <- function(
    release = c("dev", "staging", "production"),
    ...
) {
    
    # Set config environment
    Sys.setenv("R_CONFIG_ACTIVE" = release)
    
    # Load in config
    config <- config::get(file = system.file("golem-config.yml",
                                             package = "golemApp"))
    
    with_golem_options(
        app = shinyApp(
            ui = app_ui, 
            server = app_server
        ), 
        golem_opts = list(config = config)
    )
}

# app_server.R 
app_server <- function(input, output, session){
    
    # Load config
    config <- golem::get_golem_options("config")
    
    ...
}
```
