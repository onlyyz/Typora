### 一、SpdLog

#### 1 引用git库 

```c++
E:\dev\Hazel>git submodule add https://github.com/gabime/spdlog.git Hazel/vendor/spdlog
//添加包含目录
$(SolutionDir)Hazel\vendor\spdlog\include
```

![image-20230801073222968](.\assets\image-20230801073222968.png)

添加到每个项目

#### 2 创建Log 类

使用引用去检查记录器

```c++
#pragma once

#include <memory>
#include "Core.h"
#include "spdlog/spdlog.h"


namespace Hazel {

	class HAZEL_API Log
	{
	public:
		static void Init();
		//use & to check
		inline static std::shared_ptr<spdlog::logger>& GetCoreLogger() { return s_CoreLogger; }
		inline static std::shared_ptr<spdlog::logger>& GetClientLogger() { return s_ClientLogger; }
	private:
		static std::shared_ptr<spdlog::logger> s_CoreLogger;
		static std::shared_ptr<spdlog::logger> s_ClientLogger;
	
	};
}

```



```c++
//----.h
#include "Log.h"
#include "spdlog/sinks/stdout_color_sinks.h"
namespace Hazel {
    std::shared_ptr<spdlog::logger> Log::s_CoreLogger;
    std::shared_ptr<spdlog::logger> Log::s_ClientLogger;
    void Log::Init()
    {
        //time client?  value
        spdlog::set_pattern("%^[%T] &n: %v%$");
        s_CoreLogger = spdlog::stdout_color_mt("HAZEL");
        //set the log level
        s_CoreLogger->set_level(spdlog::level::trace);

        ​	s_ClientLogger = spdlog::stdout_color_mt("APP");
        ​	s_ClientLogger->set_level(spdlog::level::trace);
    }
}
```

#### 3 引擎初始化函数

```c++
#pragma once
#ifdef HZ_PLATFORM_WINDOWS

//funtion Pointer
extern Hazel::Application* Hazel::CreateApplication();
int main(int argc, char** argv) {
    Hazel::Log::Init();
    Hazel::Log::GetCoreLogger()->warn("Initialized Log!");
    Hazel::Log::GetClientLogger()->info("Hello Log!");

    auto app = Hazel::CreateApplication();
    app->Run();
    delete app;
    return 0;
}
#endif // HZ_PLATFORM_WINDOWS
```

#### 4 Log Marco

```c++
namespace Hazel {

    class HAZEL_API Log
    {
        public:
        static void Init();
        //use & to check
        inline static std::shared_ptr<spdlog::logger>& GetCoreLogger() { return s_CoreLogger; }
        inline static std::shared_ptr<spdlog::logger>& GetClientLogger() { return s_ClientLogger; }
        private:
        static std::shared_ptr<spdlog::logger> s_CoreLogger;
        static std::shared_ptr<spdlog::logger> s_ClientLogger;

    };

}

//set the marco for the log ,whcih don't use long strings
//	Core log macros
#define HZ_CORE_TRACE(...)	::Hazel::Log::GetCoreLogger()->trace(__VA_ARGS__)
#define HZ_CORE_INFO(...)	::Hazel::Log::GetCoreLogger()->info(__VA_ARGS__)
#define HZ_CORE_WARN(...)	::Hazel::Log::GetCoreLogger()->warn(__VA_ARGS__)
#define HZ_CORE_ERROR(...)	::Hazel::Log::GetCoreLogger()->error(__VA_ARGS__)
#define HZ_CORE_FATAL(...)	::Hazel::Log::GetCoreLogger()->fatal(__VA_ARGS__)

//	Client log macros
#define HZ_TRACE(...)	::Hazel::Log::GetClientLogger()->trace(__VA_ARGS__)
#define HZ_INFO(...)	::Hazel::Log::GetClientLogger()->info(__VA_ARGS__)
#define HZ_WARN(...)	::Hazel::Log::GetClientLogger()->warn(__VA_ARGS__)
#define HZ_ERROR(...)	::Hazel::Log::GetClientLogger()->error(__VA_ARGS__)
#define HZ_FATAL(...)	::Hazel::Log::GetClientLogger()->fatal(__VA_ARGS__)

//if dist build 
```

```c++

#pragma once
#ifdef HZ_PLATFORM_WINDOWS
//funtion Pointer
extern Hazel::Application* Hazel::CreateApplication();
int main(int argc, char** argv) {
    Hazel::Log::Init();
    HZ_CORE_WARN("Initialized Log!");
    HZ_INFO("Hello Log!");

    auto app = Hazel::CreateApplication();
    app->Run();
    delete app;
    return 0;
}
#endif // HZ_PLATFORM_WINDOWS
```

