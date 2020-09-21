# HockeyPlotter
[Shiny App for Plotting Shot Locations](https://aklongmuir.shinyapps.io/HockeyPlotter/)

Short explanations found on [Hockey-Graphs.com](https://hockey-graphs.com/)

Making a shot tracking application was on my agenda for this year and by mid July I had been mulling over (frustratingly attempting to teach myself) Javascript for months when I came across this tweet


[@allison_horst - What was the tool where you can draw something and it returns data (x/y coordinates) that you can be used to plot that thing? Am I dreaming? Thank you!](https://twitter.com/allison_horst/status/1285344407542050816)


And while It wasn’t exactly what I wanted to do (though I would totally play a mr scribble style game with shot locations) it was just similar enough that I finally had a jumping off point.

The UI I ended up with is pretty boring. A simple tab panel set up that gives us our input page, our data page, and then of course the density chart on the final tab. Its still very much a work in progress and there’s a lot I want to add but for now, it works. 

The Server code is where the really stuff goes down, and where I spent a lot of time trying not to cry because things didn’t quite work and I kept getting errors that the internet had no way of explaining. 

The base of the whole app is a reactiveValues object, which is then tied to a data frame where all of the data is stored. 

values <- reactiveValues()
    values$DT <- data.frame(
        x = numeric(),
        y = numeric(),
        team = factor(),
        type = factor(),
        player = factor()
    )


Next came the start of creating a plot where users can enter their data. This uses the renderPlot object but other than that is honestly just coded in ggplot like any old location chart.

output$plot1 = renderPlot({
        ggplot(values$DT, aes(x = x, y = y)) +
            coord_flip() +
            lims(x = c(0, 100), y = c(42.5, -42.5)) +
            gg_rink(spec = "nhl") +
            geom_point(
                aes(colour = factor(team), shape = factor(type)), 
                size = 5
            ) +
            theme(legend.position = "none") 

What came after this is frankly where my code gets incredibly ugly and also covered in notes so that there is some hope that in the future I will still have some idea what is going on. This is unfortunately the part where we finally use the observeEvent object. The bane of my existence while also possibly being the love of my life (its a tough gig). Essentially it works that on an input (I have ones for both a single click and a double click) the program adds a row to the data frame, taking the X and Y locations of that click, the Team selected and the player number if that is inputed and then binds that row to the existing data frame we made back at the beginning. Its also where we finally see the values$DT in the ggplot come into play as the entire code for this is kind of a self fulfilling prophesy with marks being added to the plot based on the values stored in the data table instead of being added based on the click (hence the slight lag between clicking and the mark appearing).

observeEvent(input$plot_click, {
        add_row <- data.frame(
            x = input$plot_click$y,
            y = input$plot_click$x,
            team = factor(input$Team, levels = c("Home", "Away")),
            type = "Shot",
            player = factor(input$player)
        )
        values$DT <- rbind(values$DT, add_row)

 observeEvent(input$plot_dblclick, {
        add_row <- data.frame(
            x = input$plot_dblclick$y,
            y = input$plot_dblclick$x,
            team = factor(input$Team, levels = c("Home", "Away")),
            type = "Goal",
            player = factor(input$player)
        )
        values$DT <- rbind(values$DT, add_row)

Rendering the growing table is perhaps the easiest bit of code in this entire situation, and for that, I thank it. 

output$table <- renderTable({
        values$DT

The Heatmap code is actually not that hard, it is however very annoying because it had to be written a couple of dozen time to account for all the potential filtering. Could I have done this easier with a function, well yes but that did not occur to me until literally just now and its too late frankly. The code works so that it renders based on an observed event, thus me having to code it separating for the NavBar, Team Filter and Event Filter. Look we all make mistakes. Regardless it works by isolating the values in the data table, and then producing a very nice heat map for you using the Viridis pallet option C which is the plasma pallet.

observeEvent(input$navbar,{ 
        heatmapPBP <- isolate(values$DT)
        teamfilter <- isolate(input$TeamFilter)

        heatmapPBP <- heatmapPBP %>%
            filter(team==input$TeamFilter) %>%
            filter(type %in% input$EventFilter)
      
        
        output$plot2 = renderPlot({
            ggplot(heatmapPBP, aes(x = x, y = y)) +
                coord_flip()+
                lims(x = c(0, 100), y = c(42.5, -42.5)) +
                gg_rink(spec= "nhl")+
                theme(legend.position = "none") +
                stat_density_2d(aes(fill = ..level..), geom = 'polygon', alpha = 0.4) +
                scale_fill_viridis(option = 'C')
        })
        
    })


Theres lots to still work on as I mention over at Hockey-Graphs, but hopefully this proves to be a starting point for more people being involved in manual tracking. 
