desc '上传图片(自己的目录), 并且返回地址(借用 qrsctl)'
task :put, [:key] do |task, args|
	cmd = "qrsctl put #{ENV['BUCKET']} #{args.key}.jpg ~/Pictures/com.tencent.ScreenCapture/#{args.key}.jpg"
	# puts cmd
	puts `#{cmd}`
	md_url = "![#{args.key}]({{ site.img_url }}/#{args.key}.jpg)"
	puts md_url
	`echo '#{md_url}' | pbcopy`
end