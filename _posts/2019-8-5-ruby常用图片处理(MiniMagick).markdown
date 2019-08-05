---
layout: post
title:  ruby常用图片处理(MiniMagick)
date:   2019-8-2 14:32:00
categories: ruby
---

 MiniMagick是ruby中很好用的一个图片处理库

 #### 1.resize 调整分辨率
 ```ruby
  def self.resize_image(source_path,dest_path,standard_size=[400,300])
    puts "resize_image#{source_path}"
    origin_img = MiniMagick::Image.open(source_path)
    #resize 调整大小
    origin_img.resize(standard_size.join("x"))
    origin_img.write dest_path
    origin_img.destroy!
  end
```

#### 2. 添加文字水印
```ruby
   # (x,y)表示坐标
   def self.add_text(source_path,dest_path,text,x,y,fontsize,ttf)
    origin_img = MiniMagick::Image.open(source_path)
    origin_img.combine_options do |c|
      c.pointsize fontsize #字体大小
      c.font ttf #字体文件的路径
      c.draw 'text '+x.to_s+','+y.to_s+' "'+text+'"'
    end
    origin_img.write dest_path
  end
```


#### 3. 添加透明遮罩(添加图片水印同样的逻辑)
  
  ```ruby
  #cover_path为遮罩图片的路径
  def self.add_text_and_cover(source_path,cover_path,dest_path)
    origin_img = MiniMagick::Image.open(source_path)

    cover_image = MiniMagick::Image.open(cover_path)
    origin_img1 = origin_img.composite(cover_image ) do |c|
      cover_width = origin_img.width
      cover_height = origin_img.height
      xy = "#{cover_width}x#{cover_height}"
      puts "width x height = #{xy}"
      cover_image.resize xy
      c.gravity 'center'
    end

    origin_img1.write dest_path
  end
 ``` 