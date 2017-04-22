---
title: 贪吃蛇-基于SDL
date: 2016-09-17 18:15:58
tags: ["c","linux","SDL"]
---

## 题目

贪吃蛇不用多说了吧，就是一条蛇吃水果，吃一个他的长度就加１。

## 无图无真相

![tup](/uploads/snake.png)


目前只是先实现了基本的功能，移动，长度加１。还没有实现碰撞检测的功能。


<!--more-->

```c
#include <stdio.h>
#include <stdlib.h>
#include <SDL2/SDL.h>
#include <SDL2/SDL_image.h>

#define bool int
#define true 1
#define false 0

//Screen dimension constants
#define SCREEN_WIDTH   640
#define SCREEN_HEIGHT   480
#define BOX_SIZE   20

//the map grid
#define BOX_WIDTH SCREEN_WIDTH/BOX_SIZE
#define BOX_HEIGHT   SCREEN_HEIGHT/BOX_SIZE

typedef struct _game {
	bool (*init) (struct _game * self);
	bool (*handle_events)(struct _game * self);
	bool (*update)(struct _game * self);
	bool (*render)(struct _game * self);
	bool (*close)(struct _game * self);

	//The window we'll be rendering to
	SDL_Window* gWindow;

	//The window renderer
	SDL_Renderer* gRenderer;

	//Event handler
	SDL_Event e;

	//if want to quit
	bool quit;

	//if key pressed and need update the screen
	bool need_update;

	// use to draw a snake
	int path[BOX_WIDTH][BOX_HEIGHT];
	int head_x,head_y;
	int tail_x,tail_y;

	// use to draw the fruit
	int fruit_x,fruit_y;
	int fruit_need_update;

	// counter for each node used to find the tail
	unsigned int counter;
	
} GameData, * Game;

typedef struct _color {
	Uint8 r;
	Uint8 g;
	Uint8 b;
	Uint8 a;
} Color;

bool init(struct _game * self);
bool handle_events(struct _game * self);
bool update(struct _game * self);
bool render(struct _game * self);
bool render_draw_solid_rec(Game self,int x, int y, int w, int h, Color c);
bool render_draw_empty_rec(Game self,int x, int y, int w, int h, Color c);
bool render_draw_line(Game self,int x1, int y1, int x2, int y2, Color c);
bool close(struct _game * self);

unsigned int get_count(struct _game *self)
{
	return self->counter++;
}

#define newGame(game)\
	(Game) malloc(sizeof(GameData)); \
	game->init=init;\
	game->handle_events=handle_events;\
	game->update=update;\
	game->render=render;\
	game->close=close;\
	game->gWindow=NULL;\
	game->gRenderer=NULL;\
	game->quit=false;\
	game->need_update=false;\
	game->length=3;\
	memset(game->path,0,sizeof(game->path));\
	game->counter=1;\
	game->path[10][11]=get_count(game);\
	game->path[10][12]=get_count(game);\
	game->head_x=10;game->head_y=12;\
	game->tail_x=10;game->tail_y=11;\
	game->fruit_x=rand()%BOX_WIDTH;game->fruit_y=rand()%BOX_HEIGHT;\
	game->fruit_need_update=false;

#define have_fruit()\
	(self->fruit_x!=-1 && self->fruit_y!=-1)
#define fruit_need_update()\
	(self->fruit_need_update)

bool init(Game self)
{
	//Initialization flag
	bool success = true;

	//Initialize SDL
	if( SDL_Init( SDL_INIT_VIDEO ) < 0 )
	{
		printf( "SDL could not initialize! SDL Error: %s\n", SDL_GetError() );
		success = false;
	}
	else
	{
		//Set texture filtering to linear
		if( !SDL_SetHint( SDL_HINT_RENDER_SCALE_QUALITY, "1" ) )
		{
			printf( "Warning: Linear texture filtering not enabled!" );
		}

		//Create window
		self->gWindow = SDL_CreateWindow( "Snake", SDL_WINDOWPOS_UNDEFINED, SDL_WINDOWPOS_UNDEFINED, SCREEN_WIDTH, SCREEN_HEIGHT, SDL_WINDOW_SHOWN );
		if( self->gWindow == NULL )
		{
			printf( "Window could not be created! SDL Error: %s\n", SDL_GetError() );
			success = false;
		}
		else
		{
			//Create renderer for window
			self->gRenderer = SDL_CreateRenderer( self->gWindow, -1, SDL_RENDERER_ACCELERATED );
			if(self->gRenderer == NULL )
			{
				printf( "Renderer could not be created! SDL Error: %s\n", SDL_GetError() );
				success = false;
			}
			else
			{
				//Initialize renderer color
				SDL_SetRenderDrawColor( self->gRenderer, 0xFF, 0xFF, 0xFF, 0xFF );

				//Initialize PNG loading
				int imgFlags = IMG_INIT_PNG;
				if( !( IMG_Init( imgFlags ) & imgFlags ) )
				{
					printf( "SDL_image could not initialize! SDL_image Error: %s\n", IMG_GetError() );
					success = false;
				}
			}
		}
	}

	return success;
}

bool handle_events(Game self)
{

	//Handle events on queue
	while( SDL_PollEvent( &(self->e) ) != 0 )
	{
		//User requests quit
		if( self->e.type == SDL_QUIT )
		{
			self->quit = true;
		}
		//User presses a key
        else if( self->e.type == SDL_KEYDOWN )
        {
			//Select surfaces based on key press
            switch( self->e.key.keysym.sym )
             {
                  case SDLK_UP:
				  	if (self->head_y-1!=-1&& !self->path[self->head_x][self->head_y-1])
				  	{
		                self->head_y--;	
						self->need_update=true;
				  	}
                  break;

                  case SDLK_DOWN:
				  	if (self->head_y+1!=BOX_HEIGHT && !self->path[self->head_x][self->head_y+1])
				  	{				  	
	                 	self->head_y++;
						self->need_update=true;
			  		}
                  break;

                  case SDLK_LEFT:
				  	if (self->head_x-1!=-1 && !self->path[self->head_x-1][self->head_y])
				  	{	
						self->head_x--;	
						self->need_update=true;
				  	}
				  break;

                  case SDLK_RIGHT:
				  	if (self->head_x+1!=BOX_WIDTH && !self->path[self->head_x+1][self->head_y])
				  	{	
	                     self->head_x++;
						 self->need_update=true;
				  	}
                  break;

              }

		}

	}


}

bool update(Game self)
{
	int i,j;
	if(self->need_update)
	{
		// the head always have the biggest number
		self->path[self->head_x][self->head_y]=get_count(self);

		// if head reach fruit, tail don't move
		if (self->head_x == self->fruit_x && self->head_y == self->fruit_y)
		{
			self->fruit_need_update = true;
		}
		else //the head missed fruit, need move
		{
			unsigned int max;
			//mark the tail as invisible
			self->path[self->tail_x][self->tail_y]=0;

			//find new tail, shall be min
			max=self->path[self->head_x][self->head_y];
			for (i=0;i<BOX_WIDTH;i++)
			for (j=0;j<BOX_HEIGHT;j++)
			{
				if (self->path[i][j]!=0 && self->path[i][j] < max)
				{
					self->tail_x=i;
					self->tail_y=j;
					max = self->path[i][j];
				}
			}

		}
		self->need_update=false;
	}


	if (have_fruit()&&fruit_need_update())
	{
		self->fruit_x=rand()%BOX_WIDTH;
		self->fruit_y=rand()%BOX_HEIGHT;
		self->fruit_need_update=false;
	}


	return true;
}
bool render(Game self)
{
	int i,j;
	
	//Clear screen
	SDL_SetRenderDrawColor( self->gRenderer, 0x00, 0x00, 0x00, 0x00 );
	SDL_RenderClear( self->gRenderer );

	//do the drawing...

	for (i=0;i<BOX_WIDTH;i++)
	for (j=0;j<BOX_HEIGHT;j++)
	{
		if (self->path[i][j])
		{
			Color c = {0xFF, 0x00, 0x00, 0xFF };
			render_draw_solid_rec(self,i*BOX_SIZE,j*BOX_SIZE,BOX_SIZE,BOX_SIZE,c);
		}
	}

	if (have_fruit())
	{
		Color c = {0x00, 0xFF, 0x00, 0xFF };
		render_draw_solid_rec(self,self->fruit_x*BOX_SIZE,self->fruit_y*BOX_SIZE,BOX_SIZE,BOX_SIZE,c);
	}
		
	SDL_RenderPresent( self->gRenderer );
	return true;
}

bool close(Game self)
{

	return true;
}

bool render_draw_solid_rec(Game self,int x, int y, int w, int h, Color c)
{
	SDL_Rect fillRect = { x,y,w,h };
	SDL_SetRenderDrawColor( self->gRenderer, c.r, c.g, c.b, c.a );		
	SDL_RenderFillRect( self->gRenderer, &fillRect );

}
bool render_draw_empty_rec(Game self,int x, int y, int w, int h, Color c)
{
	//Render green outlined quad
	SDL_Rect outlineRect = { x,y,w,h};
	SDL_SetRenderDrawColor( self->gRenderer, c.r, c.g, c.b, c.a );	
	SDL_RenderDrawRect( self->gRenderer, &outlineRect );

}

bool render_draw_line(Game self,int x1, int y1, int x2, int y2, Color c)
{
	//Draw blue horizontal line
	SDL_SetRenderDrawColor( self->gRenderer, c.r, c.g, c.b, c.a );	
	SDL_RenderDrawLine( self->gRenderer, x1,y1,x2,y2);

}
int main(int argc, char *argv[])
{
	Game  snake = newGame(snake) ;

	snake->init(snake);

	while(!snake->quit)
	{
		snake->handle_events(snake);
		snake->update(snake);
		snake->render(snake);
	}

	return 0;

}


```
