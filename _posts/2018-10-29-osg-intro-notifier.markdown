---
layout: post
title:  "Введение в OpenSceneGraph: Трассировка, уведомления, логирование"
date:   2018-10-29 21:50:00 +0300
categories: jekyll update
---

OpenSceneGraph имеет механизм уведомлений, позволяющий выводить отладочные сообщения в процессе выполнения рендеринга, а так же инициированные разработчиком. Это серьезное подстпорье при трассировке и отладке программы. Система уведомлений OSG поддерживает вывод диагностической информации (ошибки, предупреждения, уведомления) на уровне ядра движка и плагинов к нему. Разработчик может вывести диагностическое сообщение в процессе работы программы, воспользовавшись функцией osg::notify().

<!--more-->

Данная функция работает как стандартный поток вывода стандартной бибиотеки C++ через перегрузку оператора <<. В качестве аргумента она принимает уровень сообщения: ALWAYS, FATAL, WARN, NOTICE, INFO, DEBUG_INFO и DEBUG_FP. Например

```cpp
osg::notify(osg::WARN) << "Some warning message" << std::endl;
```
выводит предупреждение с определенным пользователем текстом.

## Перенаправление уведомлений

Уведомления OSG могут содержать важную информацию о состоянии программыЮ расширениях графической подсистемы компьютера, возможных проблемах при работе движка. 

В некоторых случаях требуется выводить эти данные не в консоль, а иметь возможность перенаправить данный вывод в файл (в виде лога) либо на любой другой интерфейс, в том числе и графический виджет. Движок содержит специальный класс osg::NotifyHandler обеспечивающий перенаправление уведомленй в нужный разработчику поток вывода.

На простом примере рассмотрим, каким образом можно перенаправить вывод уведомлений, скажем, в текстовый файл лога. Напишем следующий код

**main.h**
```cpp
#ifndef     MAIN_H
#define     MAIN_H

#include    <osgDB/ReadFile>
#include    <osgViewer/Viewer>
#include    <fstream>

#endif  //  MAIN_H
```

**main.cpp**
```cpp
#include    "main.h"

class LogFileHandler : public osg::NotifyHandler
{
public:

    LogFileHandler(const std::string &file)
    {
        _log.open(file.c_str());
    }

    virtual ~LogFileHandler()
    {
        _log.close();
    }

    virtual void notify(osg::NotifySeverity severity, const char *msg)
    {
        _log << msg;
    }

protected:

    std::ofstream   _log;
};

int main(int argc, char *argv[])
{
    osg::setNotifyLevel(osg::INFO);
    osg::setNotifyHandler(new LogFileHandler("../logs/log.txt"));

    osg::ArgumentParser args(&argc, argv);
    osg::ref_ptr<osg::Node> root = osgDB::readNodeFiles(args);

    if (!root)
    {
        OSG_FATAL << args.getApplicationName() << ": No data loaded." << std::endl;
        return -1;
    }

    osgViewer::Viewer viewer;
    viewer.setSceneData(root.get());

    return viewer.run();
}
```

Для перенаправления вывода напишем класс LogFileHandler, являющийся наследником osg::NotifyHandler. Конструктор и деструктор этого класса управляют открытием и закрытием потока вывода _log, с которым связывается текстовый файл. Метод notify() есть аналогичный петод базового класса, переопределенный нами для вывода в файл уведомлений, передаваемых OSG в процессе работы через параметр msg.

**Класс LogFileHandler**
```cpp
class LogFileHandler : public osg::NotifyHandler
{
public:

    LogFileHandler(const std::string &file)
    {
        _log.open(file.c_str());
    }

    virtual ~LogFileHandler()
    {
        _log.close();
    }

    virtual void notify(osg::NotifySeverity severity, const char *msg)
    {
        _log << msg;
    }

protected:

    std::ofstream   _log;
};
```

Далее, в основной программе выполняем необходимые настройки

```cpp
osg::setNotifyLevel(osg::INFO);
```
устанавливаем уровень уведомлений INFO, то есть вывод в лог всей информации о работе движка, включая текущие уведомления о нормальной работе.

```cpp
osg::setNotifyHandler(new LogFileHandler("../logs/log.txt"));
``` 
устанавливаем обработчик уведомлений. Далее обрабатываем аргрументы командной строки, в которых передаются пути к загружаемым моделям

```cpp
osg::ArgumentParser args(&argc, argv);
osg::ref_ptr<osg::Node> root = osgDB::readNodeFiles(args);

if (!root)
{
	OSG_FATAL << args.getApplicationName() << ": No data loaded." << std::endl;
	return -1;
}
```
При этом обрабатываем ситуацию отсутствия данных в командной строке, выводя сообщение в лог в ручном режиме посредством макроса OSG_FATAL. Запускаем программу со следующими аргументами

![](https://habrastorage.org/webt/mc/h4/so/mch4sot5pfjjpq9vb2dlolnjllm.png)

получая вывод в файл лога

```
Opened DynamicLibrary osgPlugins-3.7.0/mingw_osgdb_osgd.dll
CullSettings::readEnvironmentalVariables()
CullSettings::readEnvironmentalVariables()
Opened DynamicLibrary osgPlugins-3.7.0/mingw_osgdb_deprecated_osgd.dll
OSGReaderWriter wrappers loaded OK
CullSettings::readEnvironmentalVariables()
void StateSet::setGlobalDefaults()
void StateSet::setGlobalDefaults() ShaderPipeline disabled.
   StateSet::setGlobalDefaults() Setting up GL2 compatible shaders
CullSettings::readEnvironmentalVariables()
CullSettings::readEnvironmentalVariables()
CullSettings::readEnvironmentalVariables()
CullSettings::readEnvironmentalVariables()
ShaderComposer::ShaderComposer() 0xa5ce8f0
CullSettings::readEnvironmentalVariables()
ShaderComposer::ShaderComposer() 0xa5ce330
View::setSceneData() Reusing existing scene0xa514220
 CameraManipulator::computeHomePosition(0, 0)
    boundingSphere.center() = (-6.40034 1.96225 0.000795364)
    boundingSphere.radius() = 16.6002
 CameraManipulator::computeHomePosition(0xa52f138, 0)
    boundingSphere.center() = (-6.40034 1.96225 0.000795364)
    boundingSphere.radius() = 16.6002
Viewer::realize() - No valid contexts found, setting up view across all screens.
Applying osgViewer::ViewConfig : AcrossAllScreens
ContextData::registerGraphicsContext 0xa5d3e50
ShaderComposer::ShaderComposer() 0xa5d6228
ContextData::ContextData()0xa5d70d8
ContextData::createNewContextID() creating contextID=0
Updating the MaxNumberOfGraphicsContexts to 1
CullSettings::readEnvironmentalVariables()
  GraphicsWindow has been created successfully.0xa5d3e50
CullSettings::readEnvironmentalVariables()
CullSettings::readEnvironmentalVariables()
CullSettings::readEnvironmentalVariables()
CullSettings::readEnvironmentalVariables()
ShaderComposer::ShaderComposer() 0xa5cfe20
ShaderComposer::ShaderComposer() 0xa5d9110
ContextData::registerGraphicsContext 0xa5ddba0
ShaderComposer::ShaderComposer() 0xa5de4e0
ContextData::ContextData()0xa5dfc38
ContextData::createNewContextID() creating contextID=1
Updating the MaxNumberOfGraphicsContexts to 2
CullSettings::readEnvironmentalVariables()
  GraphicsWindow has been created successfully.0xa5ddba0
CullSettings::readEnvironmentalVariables()
CullSettings::readEnvironmentalVariables()
CullSettings::readEnvironmentalVariables()
CullSettings::readEnvironmentalVariables()
ShaderComposer::ShaderComposer() 0xa5db410
ShaderComposer::ShaderComposer() 0xa5e1328
 CameraManipulator::computeHomePosition(0, 0)
    boundingSphere.center() = (-6.40034 1.96225 0.000795364)
    boundingSphere.radius() = 16.6002
 CameraManipulator::computeHomePosition(0xa52f138, 0)
    boundingSphere.center() = (-6.40034 1.96225 0.000795364)
    boundingSphere.radius() = 16.6002
TextureObjectManager::TextureObjectManager()0xa5e5d90
osg::State::_maxTexturePoolSize=0
GLBufferObjectManager::GLBufferObjectManager()0xa5e5f50
osg::State::_maxBufferObjectPoolSize=0
GL_VENDOR = [NVIDIA_Corporation]
OpenGL extensions supported by installed OpenGL drivers are:
    GL_AMD_multi_draw_indirect
    GL_AMD_seamless_cubemap_per_texture
    GL_AMD_vertex_shader_layer
    GL_AMD_vertex_shader_viewport_index
    GL_ARB_ES2_compatibility
    GL_ARB_ES3_1_compatibility
    GL_ARB_ES3_2_compatibility
    GL_ARB_ES3_compatibility
    GL_ARB_arrays_of_arrays
    GL_ARB_base_instance
    GL_ARB_bindless_texture
    GL_ARB_blend_func_extended
    GL_ARB_buffer_storage
    GL_ARB_clear_buffer_object
    GL_ARB_clear_texture
    GL_ARB_clip_control
    GL_ARB_color_buffer_float
    GL_ARB_compatibility
    GL_ARB_compressed_texture_pixel_storage
    GL_ARB_compute_shader
    GL_ARB_compute_variable_group_size
    GL_ARB_conditional_render_inverted
    GL_ARB_conservative_depth
    GL_ARB_copy_buffer
    GL_ARB_copy_image
    GL_ARB_cull_distance
    GL_ARB_debug_output
    GL_ARB_depth_buffer_float
    GL_ARB_depth_clamp
    GL_ARB_depth_texture
    GL_ARB_derivative_control
    GL_ARB_direct_state_access
    GL_ARB_draw_buffers
    GL_ARB_draw_buffers_blend
    GL_ARB_draw_elements_base_vertex
    GL_ARB_draw_indirect
    GL_ARB_draw_instanced
    GL_ARB_enhanced_layouts
    GL_ARB_explicit_attrib_location
    GL_ARB_explicit_uniform_location
    GL_ARB_fragment_coord_conventions
    GL_ARB_fragment_layer_viewport
    GL_ARB_fragment_program
    GL_ARB_fragment_program_shadow
    GL_ARB_fragment_shader
    GL_ARB_fragment_shader_interlock
    GL_ARB_framebuffer_no_attachments
    GL_ARB_framebuffer_object
    GL_ARB_framebuffer_sRGB
    GL_ARB_geometry_shader4
    GL_ARB_get_program_binary
    GL_ARB_get_texture_sub_image
    GL_ARB_gl_spirv
    GL_ARB_gpu_shader5
    GL_ARB_gpu_shader_fp64
    GL_ARB_gpu_shader_int64
    GL_ARB_half_float_pixel
    GL_ARB_half_float_vertex
    GL_ARB_imaging
    GL_ARB_indirect_parameters
    GL_ARB_instanced_arrays
    GL_ARB_internalformat_query
    GL_ARB_internalformat_query2
    GL_ARB_invalidate_subdata
    GL_ARB_map_buffer_alignment
    GL_ARB_map_buffer_range
    GL_ARB_multi_bind
    GL_ARB_multi_draw_indirect
    GL_ARB_multisample
    GL_ARB_multitexture
    GL_ARB_occlusion_query
    GL_ARB_occlusion_query2
    GL_ARB_parallel_shader_compile
    GL_ARB_pipeline_statistics_query
    GL_ARB_pixel_buffer_object
    GL_ARB_point_parameters
    GL_ARB_point_sprite
    GL_ARB_polygon_offset_clamp
    GL_ARB_post_depth_coverage
    GL_ARB_program_interface_query
    GL_ARB_provoking_vertex
    GL_ARB_query_buffer_object
    GL_ARB_robust_buffer_access_behavior
    GL_ARB_robustness
    GL_ARB_sample_locations
    GL_ARB_sample_shading
    GL_ARB_sampler_objects
    GL_ARB_seamless_cube_map
    GL_ARB_seamless_cubemap_per_texture
    GL_ARB_separate_shader_objects
    GL_ARB_shader_atomic_counter_ops
    GL_ARB_shader_atomic_counters
    GL_ARB_shader_ballot
    GL_ARB_shader_bit_encoding
    GL_ARB_shader_clock
    GL_ARB_shader_draw_parameters
    GL_ARB_shader_group_vote
    GL_ARB_shader_image_load_store
    GL_ARB_shader_image_size
    GL_ARB_shader_objects
    GL_ARB_shader_precision
    GL_ARB_shader_storage_buffer_object
    GL_ARB_shader_subroutine
    GL_ARB_shader_texture_image_samples
    GL_ARB_shader_texture_lod
    GL_ARB_shader_viewport_layer_array
    GL_ARB_shading_language_100
    GL_ARB_shading_language_420pack
    GL_ARB_shading_language_include
    GL_ARB_shading_language_packing
    GL_ARB_shadow
    GL_ARB_sparse_buffer
    GL_ARB_sparse_texture
    GL_ARB_sparse_texture2
    GL_ARB_sparse_texture_clamp
    GL_ARB_spirv_extensions
    GL_ARB_stencil_texturing
    GL_ARB_sync
    GL_ARB_tessellation_shader
    GL_ARB_texture_barrier
    GL_ARB_texture_border_clamp
    GL_ARB_texture_buffer_object
    GL_ARB_texture_buffer_object_rgb32
    GL_ARB_texture_buffer_range
    GL_ARB_texture_compression
    GL_ARB_texture_compression_bptc
    GL_ARB_texture_compression_rgtc
    GL_ARB_texture_cube_map
    GL_ARB_texture_cube_map_array
    GL_ARB_texture_env_add
    GL_ARB_texture_env_combine
    GL_ARB_texture_env_crossbar
    GL_ARB_texture_env_dot3
    GL_ARB_texture_filter_anisotropic
    GL_ARB_texture_filter_minmax
    GL_ARB_texture_float
    GL_ARB_texture_gather
    GL_ARB_texture_mirror_clamp_to_edge
    GL_ARB_texture_mirrored_repeat
    GL_ARB_texture_multisample
    GL_ARB_texture_non_power_of_two
    GL_ARB_texture_query_levels
    GL_ARB_texture_query_lod
    GL_ARB_texture_rectangle
    GL_ARB_texture_rg
    GL_ARB_texture_rgb10_a2ui
    GL_ARB_texture_stencil8
    GL_ARB_texture_storage
    GL_ARB_texture_storage_multisample
    GL_ARB_texture_swizzle
    GL_ARB_texture_view
    GL_ARB_timer_query
    GL_ARB_transform_feedback2
    GL_ARB_transform_feedback3
    GL_ARB_transform_feedback_instanced
    GL_ARB_transform_feedback_overflow_query
    GL_ARB_transpose_matrix
    GL_ARB_uniform_buffer_object
    GL_ARB_vertex_array_bgra
    GL_ARB_vertex_array_object
    GL_ARB_vertex_attrib_64bit
    GL_ARB_vertex_attrib_binding
    GL_ARB_vertex_buffer_object
    GL_ARB_vertex_program
    GL_ARB_vertex_shader
    GL_ARB_vertex_type_10f_11f_11f_rev
    GL_ARB_vertex_type_2_10_10_10_rev
    GL_ARB_viewport_array
    GL_ARB_window_pos
    GL_ATI_draw_buffers
    GL_ATI_texture_float
    GL_ATI_texture_mirror_once
    GL_EXTX_framebuffer_mixed_formats
    GL_EXT_Cg_shader
    GL_EXT_abgr
    GL_EXT_bgra
    GL_EXT_bindable_uniform
    GL_EXT_blend_color
    GL_EXT_blend_equation_separate
    GL_EXT_blend_func_separate
    GL_EXT_blend_minmax
    GL_EXT_blend_subtract
    GL_EXT_compiled_vertex_array
    GL_EXT_depth_bounds_test
    GL_EXT_direct_state_access
    GL_EXT_draw_buffers2
    GL_EXT_draw_instanced
    GL_EXT_draw_range_elements
    GL_EXT_fog_coord
    GL_EXT_framebuffer_blit
    GL_EXT_framebuffer_multisample
    GL_EXT_framebuffer_multisample_blit_scaled
    GL_EXT_framebuffer_object
    GL_EXT_framebuffer_sRGB
    GL_EXT_geometry_shader4
    GL_EXT_gpu_program_parameters
    GL_EXT_gpu_shader4
    GL_EXT_import_sync_object
    GL_EXT_memory_object
    GL_EXT_memory_object_win32
    GL_EXT_multi_draw_arrays
    GL_EXT_packed_depth_stencil
    GL_EXT_packed_float
    GL_EXT_packed_pixels
    GL_EXT_pixel_buffer_object
    GL_EXT_point_parameters
    GL_EXT_polygon_offset_clamp
    GL_EXT_post_depth_coverage
    GL_EXT_provoking_vertex
    GL_EXT_raster_multisample
    GL_EXT_rescale_normal
    GL_EXT_secondary_color
    GL_EXT_semaphore
    GL_EXT_semaphore_win32
    GL_EXT_separate_shader_objects
    GL_EXT_separate_specular_color
    GL_EXT_shader_image_load_formatted
    GL_EXT_shader_image_load_store
    GL_EXT_shader_integer_mix
    GL_EXT_shadow_funcs
    GL_EXT_sparse_texture2
    GL_EXT_stencil_two_side
    GL_EXT_stencil_wrap
    GL_EXT_texture3D
    GL_EXT_texture_array
    GL_EXT_texture_buffer_object
    GL_EXT_texture_compression_dxt1
    GL_EXT_texture_compression_latc
    GL_EXT_texture_compression_rgtc
    GL_EXT_texture_compression_s3tc
    GL_EXT_texture_cube_map
    GL_EXT_texture_edge_clamp
    GL_EXT_texture_env_add
    GL_EXT_texture_env_combine
    GL_EXT_texture_env_dot3
    GL_EXT_texture_filter_anisotropic
    GL_EXT_texture_filter_minmax
    GL_EXT_texture_integer
    GL_EXT_texture_lod
    GL_EXT_texture_lod_bias
    GL_EXT_texture_mirror_clamp
    GL_EXT_texture_object
    GL_EXT_texture_sRGB
    GL_EXT_texture_sRGB_R8
    GL_EXT_texture_sRGB_decode
    GL_EXT_texture_shared_exponent
    GL_EXT_texture_storage
    GL_EXT_texture_swizzle
    GL_EXT_timer_query
    GL_EXT_transform_feedback2
    GL_EXT_vertex_array
    GL_EXT_vertex_array_bgra
    GL_EXT_vertex_attrib_64bit
    GL_EXT_win32_keyed_mutex
    GL_EXT_window_rectangles
    GL_IBM_rasterpos_clip
    GL_IBM_texture_mirrored_repeat
    GL_KHR_blend_equation_advanced
    GL_KHR_blend_equation_advanced_coherent
    GL_KHR_context_flush_control
    GL_KHR_debug
    GL_KHR_no_error
    GL_KHR_parallel_shader_compile
    GL_KHR_robust_buffer_access_behavior
    GL_KHR_robustness
    GL_KTX_buffer_region
    GL_NVX_blend_equation_advanced_multi_draw_buffers
    GL_NVX_conditional_render
    GL_NVX_gpu_memory_info
    GL_NVX_multigpu_info
    GL_NVX_nvenc_interop
    GL_NV_ES1_1_compatibility
    GL_NV_ES3_1_compatibility
    GL_NV_alpha_to_coverage_dither_control
    GL_NV_bindless_multi_draw_indirect
    GL_NV_bindless_multi_draw_indirect_count
    GL_NV_bindless_texture
    GL_NV_blend_equation_advanced
    GL_NV_blend_equation_advanced_coherent
    GL_NV_blend_minmax_factor
    GL_NV_blend_square
    GL_NV_clip_space_w_scaling
    GL_NV_command_list
    GL_NV_compute_program5
    GL_NV_conditional_render
    GL_NV_conservative_raster
    GL_NV_conservative_raster_dilate
    GL_NV_conservative_raster_pre_snap_triangles
    GL_NV_copy_depth_to_color
    GL_NV_copy_image
    GL_NV_depth_buffer_float
    GL_NV_depth_clamp
    GL_NV_draw_texture
    GL_NV_draw_vulkan_image
    GL_NV_explicit_multisample
    GL_NV_feature_query
    GL_NV_fence
    GL_NV_fill_rectangle
    GL_NV_float_buffer
    GL_NV_fog_distance
    GL_NV_fragment_coverage_to_color
    GL_NV_fragment_program
    GL_NV_fragment_program2
    GL_NV_fragment_program_option
    GL_NV_fragment_shader_interlock
    GL_NV_framebuffer_mixed_samples
    GL_NV_framebuffer_multisample_coverage
    GL_NV_geometry_shader4
    GL_NV_geometry_shader_passthrough
    GL_NV_gpu_program4
    GL_NV_gpu_program4_1
    GL_NV_gpu_program5
    GL_NV_gpu_program5_mem_extended
    GL_NV_gpu_program_fp64
    GL_NV_gpu_shader5
    GL_NV_half_float
    GL_NV_internalformat_sample_query
    GL_NV_light_max_exponent
    GL_NV_memory_attachment
    GL_NV_multisample_coverage
    GL_NV_multisample_filter_hint
    GL_NV_occlusion_query
    GL_NV_packed_depth_stencil
    GL_NV_parameter_buffer_object
    GL_NV_parameter_buffer_object2
    GL_NV_path_rendering
    GL_NV_path_rendering_shared_edge
    GL_NV_pixel_data_range
    GL_NV_point_sprite
    GL_NV_primitive_restart
    GL_NV_query_resource
    GL_NV_query_resource_tag
    GL_NV_register_combiners
    GL_NV_register_combiners2
    GL_NV_sample_locations
    GL_NV_sample_mask_override_coverage
    GL_NV_shader_atomic_counters
    GL_NV_shader_atomic_float
    GL_NV_shader_atomic_float64
    GL_NV_shader_atomic_fp16_vector
    GL_NV_shader_atomic_int64
    GL_NV_shader_buffer_load
    GL_NV_shader_storage_buffer_object
    GL_NV_shader_thread_group
    GL_NV_shader_thread_shuffle
    GL_NV_stereo_view_rendering
    GL_NV_texgen_reflection
    GL_NV_texture_barrier
    GL_NV_texture_compression_vtc
    GL_NV_texture_env_combine4
    GL_NV_texture_multisample
    GL_NV_texture_rectangle
    GL_NV_texture_rectangle_compressed
    GL_NV_texture_shader
    GL_NV_texture_shader2
    GL_NV_texture_shader3
    GL_NV_transform_feedback
    GL_NV_transform_feedback2
    GL_NV_uniform_buffer_unified_memory
    GL_NV_vertex_array_range
    GL_NV_vertex_array_range2
    GL_NV_vertex_attrib_integer_64bit
    GL_NV_vertex_buffer_unified_memory
    GL_NV_vertex_program
    GL_NV_vertex_program1_1
    GL_NV_vertex_program2
    GL_NV_vertex_program2_option
    GL_NV_vertex_program3
    GL_NV_viewport_array2
    GL_NV_viewport_swizzle
    GL_OVR_multiview
    GL_OVR_multiview2
    GL_S3_s3tc
    GL_SGIS_generate_mipmap
    GL_SGIS_texture_lod
    GL_SGIX_depth_texture
    GL_SGIX_shadow
    GL_SUN_slice_accum
    GL_WIN_swap_hint
    WGL_ARB_buffer_region
    WGL_ARB_context_flush_control
    WGL_ARB_create_context
    WGL_ARB_create_context_no_error
    WGL_ARB_create_context_profile
    WGL_ARB_create_context_robustness
    WGL_ARB_extensions_string
    WGL_ARB_make_current_read
    WGL_ARB_multisample
    WGL_ARB_pbuffer
    WGL_ARB_pixel_format
    WGL_ARB_pixel_format_float
    WGL_ARB_render_texture
    WGL_ATI_pixel_format_float
    WGL_EXT_colorspace
    WGL_EXT_create_context_es2_profile
    WGL_EXT_create_context_es_profile
    WGL_EXT_extensions_string
    WGL_EXT_framebuffer_sRGB
    WGL_EXT_pixel_format_packed_float
    WGL_EXT_swap_control
    WGL_EXT_swap_control_tear
    WGL_NVX_DX_interop
    WGL_NV_DX_interop
    WGL_NV_DX_interop2
    WGL_NV_copy_image
    WGL_NV_delay_before_swap
    WGL_NV_float_buffer
    WGL_NV_multisample_coverage
    WGL_NV_render_depth_texture
    WGL_NV_render_texture_rectangle
OpenGL extension 'GL_ARB_shader_objects' is supported.
OpenGL extension 'GL_ARB_vertex_shader' is supported.
OpenGL extension 'GL_ARB_fragment_shader' is supported.
OpenGL extension 'GL_ARB_shading_language_100' is supported.
OpenGL extension 'GL_EXT_geometry_shader4' is supported.
OpenGL extension 'GL_EXT_gpu_shader4' is supported.
OpenGL extension 'GL_ARB_tessellation_shader' is supported.
OpenGL extension 'GL_ARB_uniform_buffer_object' is supported.
OpenGL extension 'GL_ARB_get_program_binary' is supported.
OpenGL extension 'GL_ARB_gpu_shader_fp64' is supported.
OpenGL extension 'GL_ARB_shader_atomic_counters' is supported.
OpenGL extension 'GL_ARB_texture_rectangle' is supported.
OpenGL extension 'GL_ARB_texture_cube_map' is supported.
OpenGL extension 'GL_ARB_clip_control' is supported.
glVersion=4.6, isGlslSupported=YES, glslLanguageVersion=4.6
OpenGL extension 'GL_ARB_vertex_buffer_object' is supported.
OpenGL extension 'GL_ARB_pixel_buffer_object' is supported.
OpenGL extension 'GL_ARB_texture_buffer_object' is supported.
OpenGL extension 'GL_ARB_vertex_array_object' is supported.
OpenGL extension 'GL_ARB_transform_feedback2' is supported.
OpenGL extension 'GL_EXT_blend_func_separate' is supported.
OpenGL extension 'GL_EXT_secondary_color' is supported.
OpenGL extension 'GL_EXT_fog_coord' is supported.
OpenGL extension 'GL_ARB_multitexture' is supported.
OpenGL extension 'GL_NV_occlusion_query' is supported.
OpenGL extension 'GL_ARB_occlusion_query' is supported.
OpenGL extension 'GL_EXT_timer_query' is supported.
OpenGL extension 'GL_ARB_timer_query' is supported.
OpenGL extension 'GL_ARB_texture_multisample' is supported.
OpenGL extension 'GL_ARB_vertex_program' is supported.
OpenGL extension 'GL_ARB_fragment_program' is supported.
OpenGL extension 'GL_ARB_multitexture' is supported.
OpenGL extension 'GL_EXT_texture_filter_anisotropic' is supported.
OpenGL extension 'GL_ARB_texture_swizzle' is supported.
OpenGL extension 'GL_ARB_texture_compression' is supported.
OpenGL extension 'GL_EXT_texture_compression_s3tc' is supported.
OpenGL extension 'GL_IMG_texture_compression_pvrtc' is not supported.
OpenGL extension 'GL_OES_compressed_ETC1_RGB8_texture' is not supported.
OpenGL extension 'GL_ARB_ES3_compatibility' is supported.
OpenGL extension 'GL_EXT_texture_compression_rgtc' is supported.
OpenGL extension 'GL_IMG_texture_compression_pvrtc' is not supported.
OpenGL extension 'GL_IBM_texture_mirrored_repeat' is supported.
OpenGL extension 'GL_EXT_texture_edge_clamp' is supported.
OpenGL extension 'GL_ARB_texture_border_clamp' is supported.
OpenGL extension 'GL_SGIS_generate_mipmap' is supported.
OpenGL extension 'GL_ARB_texture_multisample' is supported.
OpenGL extension 'GL_ARB_shadow' is supported.
OpenGL extension 'GL_ARB_shadow_ambient' is not supported.
OpenGL extension 'GL_APPLE_client_storage' is not supported.
OpenGL extension 'GL_OES_texture_npot' is not supported.
OpenGL extension 'GL_ARB_texture_non_power_of_two' is supported.
OpenGL extension 'GL_EXT_texture_integer' is supported.
OpenGL extension 'GL_EXT_texture3D' is supported.
OpenGL extension 'GL_EXT_texture_array' is supported.
OpenGL extension 'GL_EXT_blend_color' is supported.
OpenGL extension 'GL_EXT_blend_equation' is not supported.
OpenGL extension 'GL_EXT_blend_equation_separate' is supported.
OpenGL extension 'GL_SGIX_blend_alpha_minmax' is not supported.
OpenGL extension 'GL_EXT_blend_logic_op' is not supported.
OpenGL extension 'GL_EXT_stencil_wrap' is supported.
OpenGL extension 'GL_EXT_stencil_two_side' is supported.
OpenGL extension 'GL_ATI_separate_stencil' is not supported.
OpenGL extension 'GL_ARB_color_buffer_float' is supported.
OpenGL extension 'GL_ARB_point_sprite' is supported.
OpenGL extension 'GL_ARB_multisample' is supported.
OpenGL extension 'GL_NV_multisample_filter_hint' is supported.
OpenGL extension 'GL_EXT_framebuffer_object' is supported.
OpenGL extension 'GL_EXT_packed_depth_stencil' is supported.
OpenGL extension 'GL_ARB_vertex_attrib_binding' is supported.
OpenGL extension 'GL_ARB_viewport_array' is supported.
OpenGL extension 'GL_EXT_secondary_color' is supported.
OpenGL extension 'GL_EXT_fog_coord' is supported.
OpenGL extension 'GL_ARB_vertex_buffer_object' is supported.
osg::State::initializeExtensionProcs() _forceVertexArrayObject = 0
                                       _forceVertexBufferObject = 0
GraphicsCostEstimator::calibrate(..)
GraphicsWindowWin32::setSyncToVBlank on
TextureObjectManager::TextureObjectManager()0xa5fe4b0
osg::State::_maxTexturePoolSize=0
GLBufferObjectManager::GLBufferObjectManager()0xa5fe5a0
osg::State::_maxBufferObjectPoolSize=0
GL_VENDOR = [NVIDIA_Corporation]
OpenGL extensions supported by installed OpenGL drivers are:
    GL_AMD_multi_draw_indirect
    GL_AMD_seamless_cubemap_per_texture
    GL_AMD_vertex_shader_layer
    GL_AMD_vertex_shader_viewport_index
    GL_ARB_ES2_compatibility
    GL_ARB_ES3_1_compatibility
    GL_ARB_ES3_2_compatibility
    GL_ARB_ES3_compatibility
    GL_ARB_arrays_of_arrays
    GL_ARB_base_instance
    GL_ARB_bindless_texture
    GL_ARB_blend_func_extended
    GL_ARB_buffer_storage
    GL_ARB_clear_buffer_object
    GL_ARB_clear_texture
    GL_ARB_clip_control
    GL_ARB_color_buffer_float
    GL_ARB_compatibility
    GL_ARB_compressed_texture_pixel_storage
    GL_ARB_compute_shader
    GL_ARB_compute_variable_group_size
    GL_ARB_conditional_render_inverted
    GL_ARB_conservative_depth
    GL_ARB_copy_buffer
    GL_ARB_copy_image
    GL_ARB_cull_distance
    GL_ARB_debug_output
    GL_ARB_depth_buffer_float
    GL_ARB_depth_clamp
    GL_ARB_depth_texture
    GL_ARB_derivative_control
    GL_ARB_direct_state_access
    GL_ARB_draw_buffers
    GL_ARB_draw_buffers_blend
    GL_ARB_draw_elements_base_vertex
    GL_ARB_draw_indirect
    GL_ARB_draw_instanced
    GL_ARB_enhanced_layouts
    GL_ARB_explicit_attrib_location
    GL_ARB_explicit_uniform_location
    GL_ARB_fragment_coord_conventions
    GL_ARB_fragment_layer_viewport
    GL_ARB_fragment_program
    GL_ARB_fragment_program_shadow
    GL_ARB_fragment_shader
    GL_ARB_fragment_shader_interlock
    GL_ARB_framebuffer_no_attachments
    GL_ARB_framebuffer_object
    GL_ARB_framebuffer_sRGB
    GL_ARB_geometry_shader4
    GL_ARB_get_program_binary
    GL_ARB_get_texture_sub_image
    GL_ARB_gl_spirv
    GL_ARB_gpu_shader5
    GL_ARB_gpu_shader_fp64
    GL_ARB_gpu_shader_int64
    GL_ARB_half_float_pixel
    GL_ARB_half_float_vertex
    GL_ARB_imaging
    GL_ARB_indirect_parameters
    GL_ARB_instanced_arrays
    GL_ARB_internalformat_query
    GL_ARB_internalformat_query2
    GL_ARB_invalidate_subdata
    GL_ARB_map_buffer_alignment
    GL_ARB_map_buffer_range
    GL_ARB_multi_bind
    GL_ARB_multi_draw_indirect
    GL_ARB_multisample
    GL_ARB_multitexture
    GL_ARB_occlusion_query
    GL_ARB_occlusion_query2
    GL_ARB_parallel_shader_compile
    GL_ARB_pipeline_statistics_query
    GL_ARB_pixel_buffer_object
    GL_ARB_point_parameters
    GL_ARB_point_sprite
    GL_ARB_polygon_offset_clamp
    GL_ARB_post_depth_coverage
    GL_ARB_program_interface_query
    GL_ARB_provoking_vertex
    GL_ARB_query_buffer_object
    GL_ARB_robust_buffer_access_behavior
    GL_ARB_robustness
    GL_ARB_sample_locations
    GL_ARB_sample_shading
    GL_ARB_sampler_objects
    GL_ARB_seamless_cube_map
    GL_ARB_seamless_cubemap_per_texture
    GL_ARB_separate_shader_objects
    GL_ARB_shader_atomic_counter_ops
    GL_ARB_shader_atomic_counters
    GL_ARB_shader_ballot
    GL_ARB_shader_bit_encoding
    GL_ARB_shader_clock
    GL_ARB_shader_draw_parameters
    GL_ARB_shader_group_vote
    GL_ARB_shader_image_load_store
    GL_ARB_shader_image_size
    GL_ARB_shader_objects
    GL_ARB_shader_precision
    GL_ARB_shader_storage_buffer_object
    GL_ARB_shader_subroutine
    GL_ARB_shader_texture_image_samples
    GL_ARB_shader_texture_lod
    GL_ARB_shader_viewport_layer_array
    GL_ARB_shading_language_100
    GL_ARB_shading_language_420pack
    GL_ARB_shading_language_include
    GL_ARB_shading_language_packing
    GL_ARB_shadow
    GL_ARB_sparse_buffer
    GL_ARB_sparse_texture
    GL_ARB_sparse_texture2
    GL_ARB_sparse_texture_clamp
    GL_ARB_spirv_extensions
    GL_ARB_stencil_texturing
    GL_ARB_sync
    GL_ARB_tessellation_shader
    GL_ARB_texture_barrier
    GL_ARB_texture_border_clamp
    GL_ARB_texture_buffer_object
    GL_ARB_texture_buffer_object_rgb32
    GL_ARB_texture_buffer_range
    GL_ARB_texture_compression
    GL_ARB_texture_compression_bptc
    GL_ARB_texture_compression_rgtc
    GL_ARB_texture_cube_map
    GL_ARB_texture_cube_map_array
    GL_ARB_texture_env_add
    GL_ARB_texture_env_combine
    GL_ARB_texture_env_crossbar
    GL_ARB_texture_env_dot3
    GL_ARB_texture_filter_anisotropic
    GL_ARB_texture_filter_minmax
    GL_ARB_texture_float
    GL_ARB_texture_gather
    GL_ARB_texture_mirror_clamp_to_edge
    GL_ARB_texture_mirrored_repeat
    GL_ARB_texture_multisample
    GL_ARB_texture_non_power_of_two
    GL_ARB_texture_query_levels
    GL_ARB_texture_query_lod
    GL_ARB_texture_rectangle
    GL_ARB_texture_rg
    GL_ARB_texture_rgb10_a2ui
    GL_ARB_texture_stencil8
    GL_ARB_texture_storage
    GL_ARB_texture_storage_multisample
    GL_ARB_texture_swizzle
    GL_ARB_texture_view
    GL_ARB_timer_query
    GL_ARB_transform_feedback2
    GL_ARB_transform_feedback3
    GL_ARB_transform_feedback_instanced
    GL_ARB_transform_feedback_overflow_query
    GL_ARB_transpose_matrix
    GL_ARB_uniform_buffer_object
    GL_ARB_vertex_array_bgra
    GL_ARB_vertex_array_object
    GL_ARB_vertex_attrib_64bit
    GL_ARB_vertex_attrib_binding
    GL_ARB_vertex_buffer_object
    GL_ARB_vertex_program
    GL_ARB_vertex_shader
    GL_ARB_vertex_type_10f_11f_11f_rev
    GL_ARB_vertex_type_2_10_10_10_rev
    GL_ARB_viewport_array
    GL_ARB_window_pos
    GL_ATI_draw_buffers
    GL_ATI_texture_float
    GL_ATI_texture_mirror_once
    GL_EXTX_framebuffer_mixed_formats
    GL_EXT_Cg_shader
    GL_EXT_abgr
    GL_EXT_bgra
    GL_EXT_bindable_uniform
    GL_EXT_blend_color
    GL_EXT_blend_equation_separate
    GL_EXT_blend_func_separate
    GL_EXT_blend_minmax
    GL_EXT_blend_subtract
    GL_EXT_compiled_vertex_array
    GL_EXT_depth_bounds_test
    GL_EXT_direct_state_access
    GL_EXT_draw_buffers2
    GL_EXT_draw_instanced
    GL_EXT_draw_range_elements
    GL_EXT_fog_coord
    GL_EXT_framebuffer_blit
    GL_EXT_framebuffer_multisample
    GL_EXT_framebuffer_multisample_blit_scaled
    GL_EXT_framebuffer_object
    GL_EXT_framebuffer_sRGB
    GL_EXT_geometry_shader4
    GL_EXT_gpu_program_parameters
    GL_EXT_gpu_shader4
    GL_EXT_import_sync_object
    GL_EXT_memory_object
    GL_EXT_memory_object_win32
    GL_EXT_multi_draw_arrays
    GL_EXT_packed_depth_stencil
    GL_EXT_packed_float
    GL_EXT_packed_pixels
    GL_EXT_pixel_buffer_object
    GL_EXT_point_parameters
    GL_EXT_polygon_offset_clamp
    GL_EXT_post_depth_coverage
    GL_EXT_provoking_vertex
    GL_EXT_raster_multisample
    GL_EXT_rescale_normal
    GL_EXT_secondary_color
    GL_EXT_semaphore
    GL_EXT_semaphore_win32
    GL_EXT_separate_shader_objects
    GL_EXT_separate_specular_color
    GL_EXT_shader_image_load_formatted
    GL_EXT_shader_image_load_store
    GL_EXT_shader_integer_mix
    GL_EXT_shadow_funcs
    GL_EXT_sparse_texture2
    GL_EXT_stencil_two_side
    GL_EXT_stencil_wrap
    GL_EXT_texture3D
    GL_EXT_texture_array
    GL_EXT_texture_buffer_object
    GL_EXT_texture_compression_dxt1
    GL_EXT_texture_compression_latc
    GL_EXT_texture_compression_rgtc
    GL_EXT_texture_compression_s3tc
    GL_EXT_texture_cube_map
    GL_EXT_texture_edge_clamp
    GL_EXT_texture_env_add
    GL_EXT_texture_env_combine
    GL_EXT_texture_env_dot3
    GL_EXT_texture_filter_anisotropic
    GL_EXT_texture_filter_minmax
    GL_EXT_texture_integer
    GL_EXT_texture_lod
    GL_EXT_texture_lod_bias
    GL_EXT_texture_mirror_clamp
    GL_EXT_texture_object
    GL_EXT_texture_sRGB
    GL_EXT_texture_sRGB_R8
    GL_EXT_texture_sRGB_decode
    GL_EXT_texture_shared_exponent
    GL_EXT_texture_storage
    GL_EXT_texture_swizzle
    GL_EXT_timer_query
    GL_EXT_transform_feedback2
    GL_EXT_vertex_array
    GL_EXT_vertex_array_bgra
    GL_EXT_vertex_attrib_64bit
    GL_EXT_win32_keyed_mutex
    GL_EXT_window_rectangles
    GL_IBM_rasterpos_clip
    GL_IBM_texture_mirrored_repeat
    GL_KHR_blend_equation_advanced
    GL_KHR_blend_equation_advanced_coherent
    GL_KHR_context_flush_control
    GL_KHR_debug
    GL_KHR_no_error
    GL_KHR_parallel_shader_compile
    GL_KHR_robust_buffer_access_behavior
    GL_KHR_robustness
    GL_KTX_buffer_region
    GL_NVX_blend_equation_advanced_multi_draw_buffers
    GL_NVX_conditional_render
    GL_NVX_gpu_memory_info
    GL_NVX_multigpu_info
    GL_NVX_nvenc_interop
    GL_NV_ES1_1_compatibility
    GL_NV_ES3_1_compatibility
    GL_NV_alpha_to_coverage_dither_control
    GL_NV_bindless_multi_draw_indirect
    GL_NV_bindless_multi_draw_indirect_count
    GL_NV_bindless_texture
    GL_NV_blend_equation_advanced
    GL_NV_blend_equation_advanced_coherent
    GL_NV_blend_minmax_factor
    GL_NV_blend_square
    GL_NV_clip_space_w_scaling
    GL_NV_command_list
    GL_NV_compute_program5
    GL_NV_conditional_render
    GL_NV_conservative_raster
    GL_NV_conservative_raster_dilate
    GL_NV_conservative_raster_pre_snap_triangles
    GL_NV_copy_depth_to_color
    GL_NV_copy_image
    GL_NV_depth_buffer_float
    GL_NV_depth_clamp
    GL_NV_draw_texture
    GL_NV_draw_vulkan_image
    GL_NV_explicit_multisample
    GL_NV_feature_query
    GL_NV_fence
    GL_NV_fill_rectangle
    GL_NV_float_buffer
    GL_NV_fog_distance
    GL_NV_fragment_coverage_to_color
    GL_NV_fragment_program
    GL_NV_fragment_program2
    GL_NV_fragment_program_option
    GL_NV_fragment_shader_interlock
    GL_NV_framebuffer_mixed_samples
    GL_NV_framebuffer_multisample_coverage
    GL_NV_geometry_shader4
    GL_NV_geometry_shader_passthrough
    GL_NV_gpu_program4
    GL_NV_gpu_program4_1
    GL_NV_gpu_program5
    GL_NV_gpu_program5_mem_extended
    GL_NV_gpu_program_fp64
    GL_NV_gpu_shader5
    GL_NV_half_float
    GL_NV_internalformat_sample_query
    GL_NV_light_max_exponent
    GL_NV_memory_attachment
    GL_NV_multisample_coverage
    GL_NV_multisample_filter_hint
    GL_NV_occlusion_query
    GL_NV_packed_depth_stencil
    GL_NV_parameter_buffer_object
    GL_NV_parameter_buffer_object2
    GL_NV_path_rendering
    GL_NV_path_rendering_shared_edge
    GL_NV_pixel_data_range
    GL_NV_point_sprite
    GL_NV_primitive_restart
    GL_NV_query_resource
    GL_NV_query_resource_tag
    GL_NV_register_combiners
    GL_NV_register_combiners2
    GL_NV_sample_locations
    GL_NV_sample_mask_override_coverage
    GL_NV_shader_atomic_counters
    GL_NV_shader_atomic_float
    GL_NV_shader_atomic_float64
    GL_NV_shader_atomic_fp16_vector
    GL_NV_shader_atomic_int64
    GL_NV_shader_buffer_load
    GL_NV_shader_storage_buffer_object
    GL_NV_shader_thread_group
    GL_NV_shader_thread_shuffle
    GL_NV_stereo_view_rendering
    GL_NV_texgen_reflection
    GL_NV_texture_barrier
    GL_NV_texture_compression_vtc
    GL_NV_texture_env_combine4
    GL_NV_texture_multisample
    GL_NV_texture_rectangle
    GL_NV_texture_rectangle_compressed
    GL_NV_texture_shader
    GL_NV_texture_shader2
    GL_NV_texture_shader3
    GL_NV_transform_feedback
    GL_NV_transform_feedback2
    GL_NV_uniform_buffer_unified_memory
    GL_NV_vertex_array_range
    GL_NV_vertex_array_range2
    GL_NV_vertex_attrib_integer_64bit
    GL_NV_vertex_buffer_unified_memory
    GL_NV_vertex_program
    GL_NV_vertex_program1_1
    GL_NV_vertex_program2
    GL_NV_vertex_program2_option
    GL_NV_vertex_program3
    GL_NV_viewport_array2
    GL_NV_viewport_swizzle
    GL_OVR_multiview
    GL_OVR_multiview2
    GL_S3_s3tc
    GL_SGIS_generate_mipmap
    GL_SGIS_texture_lod
    GL_SGIX_depth_texture
    GL_SGIX_shadow
    GL_SUN_slice_accum
    GL_WIN_swap_hint
    WGL_ARB_buffer_region
    WGL_ARB_context_flush_control
    WGL_ARB_create_context
    WGL_ARB_create_context_no_error
    WGL_ARB_create_context_profile
    WGL_ARB_create_context_robustness
    WGL_ARB_extensions_string
    WGL_ARB_make_current_read
    WGL_ARB_multisample
    WGL_ARB_pbuffer
    WGL_ARB_pixel_format
    WGL_ARB_pixel_format_float
    WGL_ARB_render_texture
    WGL_ATI_pixel_format_float
    WGL_EXT_colorspace
    WGL_EXT_create_context_es2_profile
    WGL_EXT_create_context_es_profile
    WGL_EXT_extensions_string
    WGL_EXT_framebuffer_sRGB
    WGL_EXT_pixel_format_packed_float
    WGL_EXT_swap_control
    WGL_EXT_swap_control_tear
    WGL_NVX_DX_interop
    WGL_NV_DX_interop
    WGL_NV_DX_interop2
    WGL_NV_copy_image
    WGL_NV_delay_before_swap
    WGL_NV_float_buffer
    WGL_NV_multisample_coverage
    WGL_NV_render_depth_texture
    WGL_NV_render_texture_rectangle
OpenGL extension 'GL_ARB_shader_objects' is supported.
OpenGL extension 'GL_ARB_vertex_shader' is supported.
OpenGL extension 'GL_ARB_fragment_shader' is supported.
OpenGL extension 'GL_ARB_shading_language_100' is supported.
OpenGL extension 'GL_EXT_geometry_shader4' is supported.
OpenGL extension 'GL_EXT_gpu_shader4' is supported.
OpenGL extension 'GL_ARB_tessellation_shader' is supported.
OpenGL extension 'GL_ARB_uniform_buffer_object' is supported.
OpenGL extension 'GL_ARB_get_program_binary' is supported.
OpenGL extension 'GL_ARB_gpu_shader_fp64' is supported.
OpenGL extension 'GL_ARB_shader_atomic_counters' is supported.
OpenGL extension 'GL_ARB_texture_rectangle' is supported.
OpenGL extension 'GL_ARB_texture_cube_map' is supported.
OpenGL extension 'GL_ARB_clip_control' is supported.
glVersion=4.6, isGlslSupported=YES, glslLanguageVersion=4.6
OpenGL extension 'GL_ARB_vertex_buffer_object' is supported.
OpenGL extension 'GL_ARB_pixel_buffer_object' is supported.
OpenGL extension 'GL_ARB_texture_buffer_object' is supported.
OpenGL extension 'GL_ARB_vertex_array_object' is supported.
OpenGL extension 'GL_ARB_transform_feedback2' is supported.
OpenGL extension 'GL_EXT_blend_func_separate' is supported.
OpenGL extension 'GL_EXT_secondary_color' is supported.
OpenGL extension 'GL_EXT_fog_coord' is supported.
OpenGL extension 'GL_ARB_multitexture' is supported.
OpenGL extension 'GL_NV_occlusion_query' is supported.
OpenGL extension 'GL_ARB_occlusion_query' is supported.
OpenGL extension 'GL_EXT_timer_query' is supported.
OpenGL extension 'GL_ARB_timer_query' is supported.
OpenGL extension 'GL_ARB_texture_multisample' is supported.
OpenGL extension 'GL_ARB_vertex_program' is supported.
OpenGL extension 'GL_ARB_fragment_program' is supported.
OpenGL extension 'GL_ARB_multitexture' is supported.
OpenGL extension 'GL_EXT_texture_filter_anisotropic' is supported.
OpenGL extension 'GL_ARB_texture_swizzle' is supported.
OpenGL extension 'GL_ARB_texture_compression' is supported.
OpenGL extension 'GL_EXT_texture_compression_s3tc' is supported.
OpenGL extension 'GL_IMG_texture_compression_pvrtc' is not supported.
OpenGL extension 'GL_OES_compressed_ETC1_RGB8_texture' is not supported.
OpenGL extension 'GL_ARB_ES3_compatibility' is supported.
OpenGL extension 'GL_EXT_texture_compression_rgtc' is supported.
OpenGL extension 'GL_IMG_texture_compression_pvrtc' is not supported.
OpenGL extension 'GL_IBM_texture_mirrored_repeat' is supported.
OpenGL extension 'GL_EXT_texture_edge_clamp' is supported.
OpenGL extension 'GL_ARB_texture_border_clamp' is supported.
OpenGL extension 'GL_SGIS_generate_mipmap' is supported.
OpenGL extension 'GL_ARB_texture_multisample' is supported.
OpenGL extension 'GL_ARB_shadow' is supported.
OpenGL extension 'GL_ARB_shadow_ambient' is not supported.
OpenGL extension 'GL_APPLE_client_storage' is not supported.
OpenGL extension 'GL_OES_texture_npot' is not supported.
OpenGL extension 'GL_ARB_texture_non_power_of_two' is supported.
OpenGL extension 'GL_EXT_texture_integer' is supported.
OpenGL extension 'GL_EXT_texture3D' is supported.
OpenGL extension 'GL_EXT_texture_array' is supported.
OpenGL extension 'GL_EXT_blend_color' is supported.
OpenGL extension 'GL_EXT_blend_equation' is not supported.
OpenGL extension 'GL_EXT_blend_equation_separate' is supported.
OpenGL extension 'GL_SGIX_blend_alpha_minmax' is not supported.
OpenGL extension 'GL_EXT_blend_logic_op' is not supported.
OpenGL extension 'GL_EXT_stencil_wrap' is supported.
OpenGL extension 'GL_EXT_stencil_two_side' is supported.
OpenGL extension 'GL_ATI_separate_stencil' is not supported.
OpenGL extension 'GL_ARB_color_buffer_float' is supported.
OpenGL extension 'GL_ARB_point_sprite' is supported.
OpenGL extension 'GL_ARB_multisample' is supported.
OpenGL extension 'GL_NV_multisample_filter_hint' is supported.
OpenGL extension 'GL_EXT_framebuffer_object' is supported.
OpenGL extension 'GL_EXT_packed_depth_stencil' is supported.
OpenGL extension 'GL_ARB_vertex_attrib_binding' is supported.
OpenGL extension 'GL_ARB_viewport_array' is supported.
OpenGL extension 'GL_EXT_secondary_color' is supported.
OpenGL extension 'GL_EXT_fog_coord' is supported.
OpenGL extension 'GL_ARB_vertex_buffer_object' is supported.
osg::State::initializeExtensionProcs() _forceVertexArrayObject = 0
                                       _forceVertexBufferObject = 0
GraphicsCostEstimator::calibrate(..)
GraphicsWindowWin32::setSyncToVBlank on
ViewerBase::configureAffinity() numProcessors=8
  databasePagers = 1
Viewer::startThreading() - starting threading
Viewer::startThreading() - contexts.size()=2
Making scene thread safe
Doing add
Doing add
Doing add
Doing add
Doing add
Doing add
Doing add
Doing add
Doing add
Doing add
  camera->getCameraThread()-> 0x16d30c40
  camera->getCameraThread()-> 0x16d30f60
Doing run 0x16d30c40 isRunning()=1
  gc->getGraphicsThread()->startThread() 0x16d30520
GraphicsCostEstimator::calibrate(..)
GraphicsWindowWin32::setSyncToVBlank on
ViewerBase::configureAffinity() numProcessors=8
  databasePagers = 1
Viewer::startThreading() - starting threading
Viewer::startThreading() - contexts.size()=2
Making scene thread safe
Doing add
Doing add
Doing add
Doing add
Doing add
Doing add
Doing add
Doing add
Doing add
Doing add
  camera->getCameraThread()-> 0x16d30c40
  camera->getCameraThread()-> 0x16d30f60
Doing run 0x16d30c40 isRunning()=1
  gc->getGraphicsThread()->startThread() 0x16d30520
  gc->getGraphicsThread()->startThread() 0x16d308b0
Set up threading
View::init()
ShaderComposer::~ShaderComposer() read() 0x16d308b0
ePagers = 1
Viewer::startThreading() - starting threDoing run 0x16d30520 isRunning()=0x1
DisplayListManager::DisplayListManager()0xa5e1460
Doing run 0x16d308b0 isRunning()=0x1
DisplayListManager::DisplayListManager()0x16d32260
OpenGL extension '' is not supported.
RenderStage::runCameraSetUp(osg::RenderInfo& renderInfo) OpenGL extension '' is not supported.
  gc->getGraphicsThread()->startThread() 0x16d308b0
Set up threading
View::init()
ShaderComposer::~ShaderComposer() read() 0x16d308b0
ePagers = 1
Viewer::startThreading() - starting threDoing run 0x16d30520 isRunning()=0x1
DisplayListManager::DisplayListManager()0xa5e1460
Doing run 0x16d308b0 isRunning()=0x1
DisplayListManager::DisplayListManager()0x16d32260
OpenGL extension '' is not supported.
RenderStage::runCameraSetUp(osg::RenderInfo& renderInfo) OpenGL extension '' is not supported.
Setting up osg::Camera::FRAME_BUFFER
Setting up osg::Camera::FRAME_BUFFER
Setting up osg::Camera::FRAME_BUFFER
ShaderComposer::~ShaderComposer() 0xa5d9110
ShaderComposer::~ShaderComposer() 0xa5e1328
OpenGL extension '' is not supported.
RenderStage::runCameraSetUp(osg::RenderInfo& renderInfo) 0xa5dd0e8
Setting up osg::Camera::FRAME_BUFFER
OpenGL extension '' is not supported.
RenderStage::runCameraSetUp(osg::RenderInfo& renderInfo) 0xa5e52f8
Setting up osg::Camera::FRAME_BUFFER
Viewer::~Viewer():: start destructor getThreads = 0x4
ViewerBase::stopThreading() - stopping threading
Renderer::release()
Renderer::release()
Cancelling OperationThread 0x16d30520 isRunning()=0x1
   Doing cancel 0x16d30520
exit loop 0x16d30520 isRunning()=0x1
  OperationThread::cancel() thread cancelled 0x16d30520 isRunning()=0
Cancelling OperationThread 0x16d30520 isRunning()=0
  OperationThread::cancel() thread cancelled 0x16d30520 isRunning()=0
Cancelling OperationThread 0x16d308b0 isRunning()=0x1
   Doing cancel 0x16d308b0
exit loop 0x16d308b0 isRunning()=0x1
  OperationThread::cancel() thread cancelled 0x16d308b0 isRunning()=0
Cancelling OperationThread 0x16d308b0 isRunning()=0
  OperationThread::cancel() thread cancelled 0x16d308b0 isRunning()=0
Cancelling OperationThread 0x16d30c40 isRunning()=0x1
   Doing cancel 0x16d30c40
exit loop 0x16d30c40 isRunning()=0x1
  OperationThread::cancel() thread cancelled 0x16d30c40 isRunning()=0
Cancelling OperationThread 0x16d30c40 isRunning()=0
  OperationThread::cancel() thread cancelled 0x16d30c40 isRunning()=0
Cancelling OperationThread 0x16d30f60 isRunning()=0x1
   Doing cancel 0x16d30f60
exit loop 0x16d30f60 isRunning()=0x1
  OperationThread::cancel() thread cancelled 0x16d30f60 isRunning()=0
Cancelling OperationThread 0x16d30f60 isRunning()=0
  OperationThread::cancel() thread cancelled 0x16d30f60 isRunning()=0
Viewer::stopThreading() - stopped threading.
close(0x1)0xa5d3e50
Releasing GL objects for Camera=0xa5d7ab8 _state=0xa5d5d38
Closing still viable window 0 _state->getContextID()=0
Doing delete of GL objects
DisplayListManager::deleteAllGLObjects() Not currently implemented
Done delete of GL objects
Doing discard of deleted OpenGL objects.
GLBufferObjectManager::~GLBufferObjectManager()0xa5e5f50
TextureObjectManager::~TextureObjectManager()0xa5e5d90
DisplayListManager::~DisplayListManager()0xa5e1460
ContextData::~ContextData()0xa5d70d8
close(0x1)0xa5ddba0
Releasing GL objects for Camera=0xa5dfc98 _state=0xa5ddff0
Closing still viable window 0 _state->getContextID()=0x1
Doing delete of GL objects
DisplayListManager::deleteAllGLObjects() Not currently implemented
Done delete of GL objects
Doing discard of deleted OpenGL objects.
GLBufferObjectManager::~GLBufferObjectManager()0xa5fe5a0
TextureObjectManager::~TextureObjectManager()0xa5fe4b0
DisplayListManager::~DisplayListManager()0x16d32260
ContextData::~ContextData()0xa5dfc38
Viewer::~Viewer() end destructor getThreads = 0
Destructing osgViewer::View
Destructing osg::View
ShaderComposer::~ShaderComposer() 0xa5ce330
ShaderComposer::~ShaderComposer() 0xa5ce8f0
ShaderComposer::~ShaderComposer() 0xa5d6228
close(0x1)0xa5d3e50
close(0)0xa5d3e50
ContextData::unregisterGraphicsContext 0xa5d3e50
DatabasePager::RequestQueue::~RequestQueue() Destructing queue.
DatabasePager::RequestQueue::~RequestQueue() Destructing queue.
DatabasePager::RequestQueue::~RequestQueue() Destructing queue.
DatabasePager::RequestQueue::~RequestQueue() Destructing queue.
ShaderComposer::~ShaderComposer() 0xa5de4e0
close(0x1)0xa5ddba0
close(0)0xa5ddba0
ContextData::unregisterGraphicsContext 0xa5ddba0
Done destructing osg::View
DatabasePager::RequestQueue::~RequestQueue() Destructing queue.
DatabasePager::RequestQueue::~RequestQueue() Destructing queue.
DatabasePager::RequestQueue::~RequestQueue() Destructing queue.
DatabasePager::RequestQueue::~RequestQueue() Destructing queue.
Closing DynamicLibrary osgPlugins-3.7.0/mingw_osgdb_osgd.dll
Closing DynamicLibrary osgPlugins-3.7.0/mingw_osgdb_deprecated_osgd.dll
```

