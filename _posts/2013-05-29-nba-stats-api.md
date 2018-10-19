---
title: NBA Stats API
categories:
  - Reverse Engineering
tags:
  - API
  - JSON
  - Python
---

Well it's been a while since I've updated things. In fact looking back at the last post it's from right around when basketball season started. Anyone who knows me knows I'm a basketball junkie. I spent a good part of the college and nba season wondering what sort of software could enhance my basketball addiction.

The thing I'm most interested in is trying to analyze plays. I've spent a good deal of the season just getting better myself breaking down plays but it intrigues me to think that software could split and categorize plays for me. Computer vision is a complicated topic though and it dawned on me that we get far more data already broken out and available than we used to. In addition to box scores, there's play by plays that categorize what happened and shotcharts. There are some great sites out there like [http://vorped.com/bball](http://vorped.com/bball){: target="_blank"} that are already doing a great job of aggregating this data.

I wanted to take this a step further though. I didn't want just NBA data. I wanted WNBA, D-League, NCAA Mens and NCAA Womens basketball stats. Did I mention I'm a basketball junkie? Well I started with the code on Vorped and started poking around some of the NBA sites. One in particular [http://stats.nba.com](http://stats.nba.com){: target="_blank"} interested me because they have box scores all the way back to 1946. What I noticed is using some of the screen scrape urls from Vorped i couldn't get old data like I could on the official NBA page. I started watching page loads with the developer tools in chrome and found the following URLs that seemed interesting.

```
http://stats.nba.com/stats/commonteamyears?LeagueID=00&callback=teaminfocallback
http://stats.nba.com/stats/teaminfocommon?Season=2012-13&SeasonType=Regular+Season&LeagueID=00&TeamID=1610612748
http://stats.nba.com/stats/scoreboard/?LeagueID=00&gameDate=05%2F27%2F2013&DayOffset=0
http://stats.nba.com/stats/boxscore?GameID=0041200314&RangeType=0&StartPeriod=0&EndPeriod=0&StartRange=0&EndRange=0
http://stats.nba.com/stats/playbyplay?GameID=0041200314&StartPeriod=0&EndPeriod=0
http://stats.nba.com/stats/shotchartdetail?Season=2012-13&SeasonType=Regular+Season&TeamID=0&PlayerID=202681&GameID=&Outcome=&Location=&Month=0&SeasonSegment=&DateFrom=&DateTo=&OpponentTeamID=0&VsConference=&VsDivision=&Position=&RookieYear=&GameSegment=&Period=0&LastNGames=0&ContextFilter=&ContextMeasure=FG_PCT
```

One of the things I noticed in some of these URLs was a LeagueID. It got my wondering if maybe the WNBA and D-League data was also available on stats.nba.com. So I basically just starting plugging in LeagueIDs to the first URL which spits out all teams. LeagueID of '00' is NBA and sure enough when I got to '10' it spit out the WNBA teams. '20' gave me the D-League teams. Now not all of these other leagues have the json data listed above for their games but the scoreboard url does work and using the scoreboard information to get the gamecode I could use the same xml urls being used in on the Vorped site. So for instance, with some easy guessing

```
http://www.nba.com/games/game_component/dynamic/20130528/MIAIND/pbp_all.xml
```

becomes

```
http://www.wnba.com/games/game_component/dynamic/20130527/CHIPHO/pbp_all.xml
```

And again, I got out the data I wanted. So my plan is to start pulling as much historical data as I can and see what I can see. I don't have college data yet but I think it would be real interesting to try to get that info from ESPN and be able to track players both men and women from college into their professional careers.

For a quick example of some of the data that can be pulled from the urls above check out

[https://github.com/knightsc/bbstats/blob/master/bin/teaminfo.py](https://github.com/knightsc/bbstats/blob/master/bin/teaminfo.py){: target="_blank"}

And the results

```
00 1610612755 1949 2012 Philadelphia 76ers PHI East Atlantic sixers
00 1610612766 2004 2012 Charlotte Bobcats CHA East Southeast bobcats
00 1610612749 1968 2012 Milwaukee Bucks MIL East Central bucks
00 1610612741 1966 2012 Chicago Bulls CHI East Central bulls
00 1610612739 1970 2012 Cleveland Cavaliers CLE East Central cavaliers
00 1610612738 1946 2012 Boston Celtics BOS East Atlantic celtics
00 1610612737 1949 2012 Atlanta Hawks ATL East Southeast hawks
00 1610612748 1988 2012 Miami Heat MIA East Southeast heat
00 1610612752 1946 2012 New York Knicks NYK East Atlantic knicks
00 1610612753 1989 2012 Orlando Magic ORL East Southeast magic
00 1610612751 1976 2012 Brooklyn Nets BKN East Atlantic nets
00 1610612754 1976 2012 Indiana Pacers IND East Central pacers
00 1610612765 1948 2012 Detroit Pistons DET East Central pistons
00 1610612761 1995 2012 Toronto Raptors TOR East Atlantic raptors
00 1610612764 1961 2012 Washington Wizards WAS East Southeast wizards
00 1610612746 1970 2012 Los Angeles Clippers LAC West Pacific clippers
00 1610612763 1995 2012 Memphis Grizzlies MEM West Southwest grizzlies
00 1610612740 1988 2012 New Orleans Hornets NOH West Southwest hornets
00 1610612762 1974 2012 Utah Jazz UTA West Northwest jazz
00 1610612758 1948 2012 Sacramento Kings SAC West Pacific kings
00 1610612747 1948 2012 Los Angeles Lakers LAL West Pacific lakers
00 1610612742 1980 2012 Dallas Mavericks DAL West Southwest mavericks
00 1610612743 1976 2012 Denver Nuggets DEN West Northwest nuggets
00 1610612745 1967 2012 Houston Rockets HOU West Southwest rockets
00 1610612759 1976 2012 San Antonio Spurs SAS West Southwest spurs
00 1610612756 1968 2012 Phoenix Suns PHX West Pacific suns
00 1610612760 1967 2012 Oklahoma City Thunder OKC West Northwest thunder
00 1610612750 1989 2012 Minnesota Timberwolves MIN West Northwest timberwolves
00 1610612757 1970 2012 Portland Trail Blazers POR West Northwest blazers
00 1610612744 1946 2012 Golden State Warriors GSW West Pacific warriors
00 1610610024 1947 1954 Baltimore Bullets BAL East  bullets
00 1610610036 1946 1950 Washington Capitols WAS   capitols
00 1610610034 1946 1949 St. Louis Bombers BOM   bombers
00 1610610025 1946 1949 Chicago Stags CHS Central  stags
00 1610610037 1949 1949 Waterloo Hawks WAT   hawks
00 1610610027 1949 1949 Denver Nuggets DN   nuggets
00 1610610030 1949 1952 Indianapolis Olympians INO West  olympians
00 1610610023 1949 1949 Anderson Packers AND West  packers
00 1610610033 1949 1949 Sheboygan Redskins SHE West  redskins
00 1610610028 1946 1946 Detroit Falcons DEF   falcons
00 1610610035 1946 1946 Toronto Huskies HUS   huskies
00 1610610031 1946 1946 Pittsburgh Ironmen PIT   ironmen
00 1610610026 1946 1947 Cleveland Rebels CLR   rebels
00 1610610032 1946 1948 Providence Steamrollers PRO   steamrollers
00 1610610029 1948 1948 Indianapolis Jets JET   jets
10 1611661330 2008 2013 Atlanta Dream ATL East  dream
10 1611661329 2006 2013 Chicago Sky CHI East  sky
10 1611661328 2000 2013 Seattle Storm SEA West  storm
10 1611661325 2000 2013 Indiana Fever IND East  fever
10 1611661324 1999 2013 Minnesota Lynx MIN West  lynx
10 1611661323 1999 2013 Connecticut Sun CON East  sun
10 1611661322 1998 2013 Washington Mystics WAS East  mystics
10 1611661321 1998 2013 Tulsa Shock TUL West  shock
10 1611661320 1997 2013 Los Angeles Sparks LAS West  sparks
10 1611661319 1997 2013 San Antonio Silver Stars SAN West  silverstars
10 1611661317 1997 2013 Phoenix Mercury PHO West  mercury
10 1611661313 1997 2013 New York Liberty NYL East  liberty
10 1611661314 1997 2006 Charlotte Sting CHA East  sting
10 1611661326 2000 2002 Miami Sol MIA East  sol
10 1611661318 1997 2009 Sacramento Monarchs SAC West  monarchs
10 1611661316 1997 2008 Houston Comets HOU West  comets
10 1611661315 1997 2003 Cleveland Rockers CLE East  rockers
10 1611661327 2000 2002 Portland Fire POR West  fire
20 1612709889 2003 2012 Tulsa 66ers TUL  Central 66ers
20 1612709917 2009 2012 Springfield Armor SPG  East armor
20 1612709913 2008 2012 Erie BayHawks ERI  East bayhawks
20 1612709914 2008 2012 Reno Bighorns RNO  West bighorns
20 1612709893 2003 2012 Canton Charge CTN  East charge
20 1612709905 2006 2012 Los Angeles D-Fenders LAD  West dfenders
20 1612709911 2007 2012 Iowa Energy IWA  Central energy
20 1612709900 2006 2012 Bakersfield Jam BAK  West jam
20 1612709918 2010 2012 Texas Legends TEX  Central legends
20 1612709910 2007 2012 Fort Wayne Mad Ants FWN  East mad_ants
20 1612709915 2009 2012 Maine Red Claws MNE  East red_claws
20 1612709904 2006 2012 Sioux Falls Skyforce SXF  Central skyforce
20 1612709903 2006 2012 Idaho Stampede IDA  West stampede
20 1612709890 2003 2012 Austin Toros AUS  Central toros
20 1612709908 2007 2012 Rio Grande Valley Vipers RGV  Central vipers
20 1612709902 2006 2012 Santa Cruz Warriors SCW  West dwarriors
20 1612709899 2006 2008 Anaheim Arsenal ANA  West arsenal
20 1612709901 2006 2008 Colorado 14ers COL  Southwest 14ers
20 1612709909 2007 2010 Utah Flash UTA West  flash
20 1612709898 2005 2006 Arkansas RimRockers ARK East  rimrockers
20 1612709897 2004 2006 Fort Worth Flyers FTW East  flyers
20 1612709891 2003 2005 Fayetteville Patriots FAY   patriots
20 1612709895 2003 2005 Florida Flame FLA   flame
20 1612709896 2003 2005 Roanoke Dazzle ROA   dazzle
20 1612709892 2003 2003 Greenville Groove GRN   None
20 1612709894 2003 2003 Mobile Revelers MOB   None
```
