# How to accelerate video trancoding (FFmpeg) in Mastodon

Let's build a remote video transcoder (FFmpeg) for Mastodon!

## Why?
Whenever there were incidents or similar events on X (formerly, Twitter), such as on [July 1st](https://apnews.com/article/twitter-outage-musk-complaints-restrictions-b59ef586491891fdd3d6c8220ba2ec0d), the Sidekiq queue in Mastodon would expand, resulting in latency exceeding 10 hours. However, no matter how many photos or videos were being exchanged, it seemed unreasonable in a network where text communication was the primary focus. Eventually, when I disabled all multimedia processing-related features, the latency dropped to zero.

## Before following this tutorial

Please make sure to optimize your Sidekiq processes and threads as much as possible before proceeding with this tutorial. For detailed information, please refer to the link below.

* How to make Sidekiq faster https://github.com/mastodon/mastodon/discussions/19797

## Edit the transcoder and image extractor

1. Get a VPS like a [Vultr](https://www.vultr.com/?ref=8255151) it will be used to run FFmpeg. Or contact a Docker/Kubernetes supported cloud provider.

2. Create methods in the file `lib/paperclip/transcoder.rb`

```ruby
    def receive_file(destination)
      puts "Downloading the file... (receive_file in lib/paperclip/transcoder.rb)"
      command = Terrapin::CommandLine.new('curl', '-k -X POST -F :source :endpoint -o :destination')
      puts command.command(destination: destination.path, endpoint: ENV['FFMPEG_API_ENDPOINT'] + '/video/extract/images?download=yes', source: 'file=@' + @file.path)
      command.run(destination: destination.path, endpoint: ENV['FFMPEG_API_ENDPOINT'] + '/video/extract/images?download=yes', source: 'file=@' + @file.path, logger: Paperclip.logger)
    end

    def receive_mp4_file(destination)
      puts 'Converting the file... (receive_mp4_file in lib/paperclip/transcoder.rb)', @file.path
      command = Terrapin::CommandLine.new('curl', '-k -X POST -F :source :endpoint -o :destination')
      puts command.command(source: 'file=@' + @file.path, destination: destination.path, endpoint: ENV['FFMPEG_API_ENDPOINT'] + '/convert/video/to/mp4')
      command.run(source: 'file=@' + @file.path, destination: destination.path, endpoint: ENV['FFMPEG_API_ENDPOINT'] + '/convert/video/to/mp4', logger: Paperclip.logger)
    end
```

3. Add this code into the file `lib/paperclip/transcoder.rb` nearby the `destination` defined

```ruby
      #@input_options  = @convert_options[:input]&.dup  || {}   # -- nearby first

      # myself
      if ENV['FFMPEG_API_ENDPOINT'].present?
        begin
          case @format.to_s
          when /jpg$/, /jpeg$/, /png$/, /gif$/
            receive_file(destination)
            return destination
          when 'mp4'
            receive_mp4_file(destination)
            return destination
          end
        rescue Terrapin::ExitStatusError => e
          raise Paperclip::Error, "Error while transcoding #{@basename}: #{e}"
        rescue Terrapin::CommandNotFoundError
          raise Paperclip::Errors::CommandNotFoundError, 'Could not run the `curl` command. Please install curl.'
        end
      end

      #case @format.to_s  # -- nearby end
```

4. Modify `extract_image_from_file` function and create methods in the file `lib/paperclip/image_extractor.rb`

```ruby
    def extract_image_from_file!
      dst = Tempfile.new([File.basename(@file.path, '.*'), '.png'])
      dst.binmode

      begin
        if ENV['FFMPEG_API_ENDPOINT'].present?
          receive_file(dst)
        else
          command = Terrapin::CommandLine.new('ffmpeg', '-i :source -loglevel :loglevel -y :destination', logger: Paperclip.logger)
          command.run(source: @file.path, destination: dst.path, loglevel: 'fatal')
        end
      rescue Terrapin::ExitStatusError
        dst.close(true)
        return nil
      rescue Terrapin::CommandNotFoundError
        if ENV['FFMPEG_API_ENDPOINT'].present?
          raise Paperclip::Errors::CommandNotFoundError, 'Could not run the `curl` command. Please install curl.'
        else
          raise Paperclip::Errors::CommandNotFoundError, 'Could not run the `ffmpeg` command. Please install ffmpeg.'
        end
      end

      dst
    end

    def receive_file(destination)
      puts "Downloading the file... (receive_file in lib/paperclip/image_extractor.rb)"
      command = Terrapin::CommandLine.new('curl', '-k -X POST -F :source :endpoint -o :destination')
      puts command.command(destination: destination.path, endpoint: ENV['FFMPEG_API_ENDPOINT'] + '/video/extract/images?download=yes', source: 'file=@' + @file.path)
      command.run(destination: destination.path, endpoint: ENV['FFMPEG_API_ENDPOINT'] + '/video/extract/images?download=yes', source: 'file=@' + @file.path, logger: Paperclip.logger)
    end
  end
```

5. (Not recommended) `ffprobe` over HTTP API - Although there is an API endpoint corresponding to `ffprobe`, I do not recommend applying it to Mastodon as it caused errors when implemented. Modify `ffmpeg_command_output` function and create methods in the file `app/lib/video_metadata_extractor.rb` like this.

```ruby
  def ffmpeg_command_output
    #if ENV['FFMPEG_API_ENDPOINT'].present?
    #  puts 'Uploading the file... (ffmpeg_command_output in app/lib/video_metadata_extractor.rb)', @path
    #  command = Terrapin::CommandLine.new('curl', '-X POST -F :path :endpoint')
    #  puts command.command(path: 'file=@' + @path, endpoint: ENV['FFMPEG_API_ENDPOINT'] + '/probe')
    #  command.run(path: 'file=@' + @path, endpoint: ENV['FFMPEG_API_ENDPOINT'] + '/probe')
    #else
    #  command = Terrapin::CommandLine.new('ffprobe', '-i :path -print_format :format -show_format -show_streams -show_error -loglevel :loglevel')
    #  command.run(path: @path, format: 'json', loglevel: 'fatal')
    #end
    command = Terrapin::CommandLine.new('ffprobe', '-i :path -print_format :format -show_format -show_streams -show_error -loglevel :loglevel')
    command.run(path: @path, format: 'json', loglevel: 'fatal')
  end
```

## Edit `.env.production`
Set the FFmpeg endpoint URL (`FFMPEG_API_ENDPOINT` variable) to `.env.production`

```
FFMPEG_API_ENDPOINT=http://transcoder-1.catswords.net:80
#FFMPEG_API_ENDPOINT=https://transcoder-1.catswords.net:443
```

## Run your own FFMPEG API server
* https://github.com/gnh1201/ffmpeg-api (Tested on Mastodon 4.2.5)

## (Optional) Disable BlurHash
FFmpeg is used for calculating [BlurHash](https://github.com/woltapp/blurhash) (in this case, invoked by ImageMagick's convert), and pre-calculating BlurHash and embedding it statically can be a solution to reduce the required computational load.

* https://blurred.dev/ - BlurHash Calculator Online (Next.js Image blurDataURL generator)

Here is an example code of the file `lib/paperclip/blurhash_transcoder.rb`:

```ruby
# frozen_string_literal: true

module Paperclip
  class BlurhashTranscoder < Paperclip::Processor
    def make
      return @file unless options[:style] == :small || options[:blurhash]

      #pixels   = convert(':source -depth 8 RGB:-', source: File.expand_path(@file.path)).unpack('C*')
      #geometry = options.fetch(:file_geometry_parser).from_file(@file)

      #attachment.instance.blurhash = Blurhash.encode(geometry.width, geometry.height, pixels, **(options[:blurhash] || {}))

      # Generated by https://blurred.dev/
      attachment.instance.blurhash = '|22PC@?q%dofM*IFM$V]t5?;?q%dofM*M$M$V]t5.4.4%JofRUM$RSWCoe%cx?tPoLV^RSRlWCj?oxoeodj@afWCWCWCWCRkRkWCayj?k9bFWCWCM}M}Rkayk9ock9WVWCM}RRRkWCk9ock9ayWCRkRkV]WCbFk9j?bFay'

      @file
    end
  end
end
```

## Use Cases
* https://github.com/gnh1201/topic-activitypub

## (TMI) Image Conversion
It is determined that image files do not have a significant impact on server performance. However, as the number of users increases, it could potentially have an impact in the future. Therefore, it is necessary to consider modifying the ffmpeg-api or referring to similar projects, as shown below.

* https://github.com/FPurchess/preview-service
* https://github.com/algoo/preview-generator

## (TMI) Other applications
Applications such as Misskey and Pleroma, which utilize the ActivityPub network, can also employ similar methods. If you wish, please contact me through my profile. I can prepare presets tailored to your application on my transcoding server.

## Original article
* https://gist.github.com/gnh1201/1ba49e0e80a11237038900bf8abfa434

## Contact me
* ActivityPub [@gnh1201@catswords.social](https://catswords.social/@gnh1201)
* abuse@catswords.net
