1. Setup a SQL database
    - -> Store posts, metadata, sentiment, ...
    - PostgreSQL in Docker Container
    - Using the PostGIS plugin to handle geographic data
      - e.g. shapes of countries, locations of cities, ...

2. Create a Table to store the posts, results from the NER/linkning, and results from the sentiment analysis
   - contains the posts themselves,
   - the found QIDs (wikidata IDs of the entities found in the post)
   - and the sentiment classification of a post
   - ![](.\screenshot posts table.png)

3. Setup Entity Fishing
    - Another docker container that runs the entity-fishing api

4. Read the reddit-data
   - line by line, each line is one post
   - use the 'normalizedBody' to run the entity-fishing model for each post
     - using a spacy wrapper that is configured to access the docker container
   - the model returns a list of entities with their labels (Person, GPE, ...) and QID
   - only further consider posts that return at least one GPE
   - store these posts in the database along with the QIDs of the found GPEs
   - we parallelized this step by running 16 workers at the same time
   - this way we processed about 70% of the dataset
   - however the process slowed has down a lot a at this point
     - probably because the database grew too large and we ran out of ram
     - running the database and the entity-fishing model in docker at the same time requires a lot of ram

5. Run sentiment analysis
   - For every post in the database (only the ones that contain a GPE)
   - Run them through the sentiment model
   - Store the results also in the posts database table

6. Create a table to store metadata about the found GPEs
   - contains the name of the GPE,
   - the type (city, country or state),
   - the border shape for countries&states / center point for cities
   ![](.\screenshot gpes table.png)

7. Annotate and filter all found GPEs with metadata
   - Query the wikidata knowledge base with the SPARQL endpoint for every GPE
   - check the 'instanceOf' attribute to classify the GPE
     - city, state or country
     - or it has been misclassified by the model and is something different
     - difficult because data is heterogeneous
     - e.g. used values for the 'instanceOf' attribute for cities ['city', 'big city', 'million city', 'largest city', 'cycling city', 'city or town','capital city', 'component city', 'city in Ukraine','megacity']
   - get additionally metadata: population, country_code, shape for state, center for cities
   - the shapes for countries is not from wikidata but from [here](https://public.opendatasoft.com/explore/dataset/world-administrative-boundaries/export/)
      - matched with the country code
   - the data is inserted into the table

8. Aggregate emotions
   - For each GPE in the database look up all the posts that contain this GPE
   - Average the sentiments for all the matching posts
   - Store the resulting average in the gpes table in the database

9. Setup Backend
   - Python backend using fastAPI
   - Queries the data from the database and sends it to the frontend
   - One endpoint for getting the averaged sentiment of either countries, cities, or state
   - Another endpoint for counting the number of post for every country/state and returning the distribution

10. Frontend
    - Build using angular
    - The map is realized via leaflet
    - The views: distribution, sentiment
      - Distribution: Coropleth map
      - Sentiemnt: Each emotion is mapped to a emoji. Top three are displayed on map